# I2C communication on STM32 MCU with HAL

This document explains how to develop a driver in C for a device/chip that is 
connected to the I2C bus of a STM32 Cortex-M MCU.

I2C is a serial communication protocol on a 2 wire bus. This means that each device
uses 2 of its pins to onnect to the I2C bus. The pins are names SCL
and SDA. obviously the device and the MCU have to share the ground too, and the
device needs power. before powering the device make sure to check its tolerated
voltage level in its datasheet.

At least one of the devices connected to the bus is the bus master. The other devices are
slaves. communication is always initiated by the master.

each device on the bus has a unique 7-bit address. so there can be 127 slaves and
one master on an I2C bus.

## I2C device programmers model

the device has a number of registers. reading and writing to these registers
impacts what the device does. so to use the device one needs to read/write to
these registers. 
each register in the device has its own address. this is usually an 8 bit address.
the registers themselves can hold 8 or 16 bits of data depending on the device.

## I2C operation

The transaction on the bus is started through a start (ST) signal. A start
condition is defined as a high to low transition on the data line while the SCL
line is held high. After the master has transmitted this, the bus is considered
busy. The next data byte transmitted after the start condition contains the
address of the slave in the first 7 bits and the eighth (lsb) bit tells whether
the master is reading from a register or writing to a register (1 for read, 0 for
write). When an address is sent, each device in the system compares the first
seven bits after a start condition with its address. If they match, the device
considers itself addressed by the master. 

Data transfer with acknowledge is mandatory. The transmitter must release the SDA
line during the acknowledge pulse. The receiver must then acknowledge by pulling
SDA low so that it remains stable low during the high period of the acknowledge
clock pulse. A receiver which has been addressed is obliged to generate an acknowledge
after each byte of data received.

So to summarize the master intitiates the communication by sending the start
condition, and then sending a slave address on SDA, 1 bit per clock cycle. 8 bits
of address are sent in total (7-bit address + 1 read/write bit). then the slave
sends an ack in the next clock cycle.

In the beginning the mester always wants to writes to the device, to send the
address of the register. so the first read/write bit is always 0. Then the master
sends an 8-bit subaddress which is the address of a register in the device, and
the slave acknowledges. 

At this point, if the master wants to write to that register, it sends the data
to write byte by byte and the slave acknowledges after each byte. in the end the
master sends a stop condition, which is a low to high transition on SDA while SCL
is held high.

If the master wants to read from the register, it first has to repeat the start
condition + slave address + read/write bit, this time with the read/write bit set
to 1. the slave acknowledges. then the slave sends the data to be read, byte by
byte. the master acknowledges after each byte. except after the last byte, when
the master sends a stop condition instead.

This operation can be checked with a logic analyzer.

## Enabling an I2C interface in the STM32 MCU using HAL functions

1. void HAL_GPIO_Init(GPIO_TypeDef  *GPIOx, GPIO_InitTypeDef *GPIO_Init)

Initialize each of the gpio pins (SDA, SCL)

example:
    /* init the SCL pin: assuming that I2C1 is used, and its scl is pin 6 of port B  */
    GPIO_InitTypeDef gpioInit;
    /* for safety */
    MEM_SET(&gpioInit, 0U, sizeof(gpioInit));
    
    gpioInit.Pin       = GPIO_PIN_6;
    gpioInit.Mode      = GPIO_MODE_AF_OD;
    gpioInit.Pull      = GPIO_NOPULL;
    gpioInit.Speed     = GPIO_SPEED_FREQ_VERY_HIGH;
    gpioInit.Alternate = GPIO_AF4_I2C1;
    
    HAL_GPIO_Init(CONFIG_I2C_SCL_GPIO_PORT , &gpioInit);

do the same for the SDA pin.

2. __HAL_RCC_GPIOB_CLK_ENABLE()

Enable the clock of the gpio port B (assuming that SDA, SCL pins are pins of port B).
if the pins are in different ports then the clocks of both ports must
be enabled.

3. HAL_RCC_I2C1_CLK_ENABLE()

Enable the clock of I2C1.
    
4. HAL_I2C_Init(I2C_HandleTypeDef *hI2C)

Initializes the I2C according to the specified parameters in the handle to I2C
and initialize the associated handle.

example:
    /* the handle to the I2C interface needs to be static and global, because it is used for each communication call */
    static I2C_HandleTypeDef I2CHandle;
    
    /* not really needed, because uninitialized global variables are normally set to zero in reset handler, but who knows */
    MEM_SET(&I2CHandle, 0U, sizeof(i2cHandle));
    
    I2CHandle.Instance             = I2C1;
    I2CHandle.Init.Timing          = 0x00F02B86; 
    I2CHandle.Init.OwnAddress1     = 0x02;   /* don't know what this is exactly */
    I2CHandle.Init.AddressingMode  = I2C_ADDRESSINGMODE_7BIT;
    I2CHandle.Init.DualAddressMode = I2C_DUALADDRESS_DISABLE;
    I2CHandle.Init.GeneralCallMode = I2C_GENERALCALL_DISABLE;
    I2CHandle.Init.NoStretchMode   = I2C_NOSTRETCH_DISABLE;

    HALStatus = HAL_I2C_Init(&I2CHandle);    

The timing constatnt specifies the I2C_TIMINGR_register value. This parameter is calculated by referring to I2C initialization section in Reference manual, or by checking a stm reference implementation.

Note that there are many more parameters in the I2C handle, that can be configured
if needed.

At this point the I2C interface is ready to use.

4. If you need to use I2C in an interrup driven manner, then enable its interrupts.

I2C error interrupt 

HAL_NVIC_SetPriority(I2C1_ER_IRQn, uint32_t PreemptPriority, uint32_t SubPriority);
HAL_NVIC_EnableIRQ(I2C1_ER_IRQn);

Same thing for the I2C event interrupt

HAL_NVIC_SetPriority(I2C1_EV_IRQn, uint32_t PreemptPriority, uint32_t SubPriority);
HAL_NVIC_EnableIRQ(I2C1_EV_IRQn);

You can then write the interrupt handlers to handle I2C errors and events.

        
5. Reading and writing to device registers

IMPORTANT: the DevAddress in the following functions is the 7-bit address of the device, shifted 1 bit to the left.

HAL_I2C_Mem_Read(I2C_HandleTypeDef *hI2C, uint16_t DevAddress, uint16_t MemAddress, uint16_t MemAddSize, uint8_t *pData, uint16_t Size, uint32_t Timeout)

example: read 1 byte of data from register 0x10

    #define LPS28DFW_I2C_ADDRESS    (0x5C << 1) /* 7-bit device address shifted 1 bit to the left */
    uint16_t registerAddress = 0x10;
    uint8_t dataIn;
    uint8_t *pReceiveData = &dataIn;
    
    HAL_I2C_Mem_Read(&I2CHandle, LPS28DFW_I2C_ADDRESS, registerAddress, I2C_MEMADD_SIZE_8BIT, pReceiveData, (uint16_t)sizeof(uint8_t), 25)

I2C_MEMADD_SIZE_8BIT is defined by HAL for device registers that are 8 bits wide.
Timeout is a duration in terms of systicks.

HAL_StatusTypeDef HAL_I2C_Mem_Write(I2C_HandleTypeDef *hI2C, uint16_t DevAddress, uint16_t MemAddress, uint16_t MemAddSize, uint8_t *pData, uint16_t Size, uint32_t Timeout)
                                    
example: write 1 byte of data to register 0x10

    uint8_t dataOut = 0x23;
    uint8_t *pTransmitData = &dataOut;
    
HAL_I2C_Mem_Write(&I2CHandle, LPS28DFW_I2C_ADDRESS, registerAddress, I2C_MEMADD_SIZE_8BIT, (uint8_t *)pTransmitData, (uint16_t)sizeof(uint8_t), 25))
                                    
In case communication doesn't work the first check your code, then check the
slave device datasheet.

