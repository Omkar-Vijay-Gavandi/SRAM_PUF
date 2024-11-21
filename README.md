# SRAM_PUF


## Task 1:- Implementation of I2C protocol in PSOC Creator:-


Schematic:-

![image](https://github.com/user-attachments/assets/ceeece6c-a4f2-4432-a8ed-2e1b10eda566)


Code:-

``` bash
#include <device.h>
#include <stdio.h>

/* Function Prototypes */
uint8 I2C_MasterWriteToEZI2C(uint8 i2CAddr, uint8 nbytes);

/* Constants */
#define EZI2C_SLAVE_ADDR (8u)
#define EZI2C_SIZE       (4u)
/* Sets the boundary between the read/write and read only areas.
*  The read/write area is first, followed by the read only area. */
#define EZI2C_RW_SIZE    (2u)

/* Data Structure Definitions */
/* I2C master data, packed for 32-bit CPU */
#if CY_PSOC3
#else
#define MASTER_BUF __attribute__ ((packed)) /* PSoC 5 */
#endif
typedef struct
{
    uint8 writeOffset; /* 1 byte because EZI2C Sub-address is 8 bits */
    uint8 i2cMasterBuffer[EZI2C_SIZE];
} MASTER_BUF EZI2C_DATASTRUCT;

/* Variables */
/* EZI2C buffer */
uint8 ezi2cBuffer[EZI2C_SIZE] = {50, 70, 90, 100};
/* master buffer */
EZI2C_DATASTRUCT i2cMasterData = {0, {0, 0, 0, 0}};

int main(void)
{
    //uint8 buttons;         /* copy of Buttons status reg */
    //uint8 dataValue;       /* temporary variable to highlight read and write operations */
    uint8 i;               /* general purpose counter */
    uint8 volatile status; /* copy of EZI2C activity status */

    /* Enable global interrupts -
    *  the EZI2C and I2C_Master Components use interrupts */
    CyGlobalIntEnable;

    /* initialize EZI2C and its buffer */
    EZI2C_Start();
    /* EZI2C buffer parameters:
    *  EZI2C_SIZE - is the size of the memory exposed to the I2C interface.
    *  EZI2C_RW_SIZE - sets the boundary between the read/write and read only areas.
    *                  The read/write area is first, followed by the read only area.
    *  (void *)ezi2cBuffer - is the pointer to the memory exposed to the I2C interface. */
    EZI2C_SetBuffer1(EZI2C_SIZE, EZI2C_RW_SIZE, (void *)ezi2cBuffer);

    /* initialize I2C master */
    I2CM_Start();
    UART_Start();
    
    UART_PutString(" Initialised the psoc");

    /* main loop */
    for(;;)
    {
        /*************************************************************************************/
        /* Do master side tasks, with the I2C Master Component, with error handling          */
        /*************************************************************************************/
        /* SW3: I2C Master updates EZI2C data (RW portion only) */
    
        /* Read the data from the EZI2C */
        /* First, write the offset byte. This is the first byte in the global
        *  structure i2cMasterData. That is, i2cMasterData.writeOffset. */
        status = I2C_MasterWriteToEZI2C(EZI2C_SLAVE_ADDR, 1);
        
        UART_PutString("Offset value is written by master");
        if(!(status & I2CM_MSTAT_ERR_XFER))
        {
            /* Then do the read */
            status = I2CM_MasterClearStatus();
            
            
            if(!(status & I2CM_MSTAT_ERR_XFER))
            {
                status = I2CM_MasterReadBuf(EZI2C_SLAVE_ADDR,
                                           (uint8 *)&(i2cMasterData.i2cMasterBuffer),
                                           EZI2C_SIZE, I2CM_MODE_COMPLETE_XFER);
                
                UART_PutString("The master reads the EZI2C buffer into its local buffer");
                
                
                if(status == I2CM_MSTR_NO_ERROR)
                {
                    /* wait for read complete and no error */
                    do
                    {
                        status = I2CM_MasterStatus();
                    } while((status & (I2CM_MSTAT_RD_CMPLT | I2CM_MSTAT_ERR_XFER)) == 0u);
                    if(!(status & I2CM_MSTAT_ERR_XFER))
                    {
                        
                        
                        /* Decrement all RW bytes in the EZI2C buffer, by different values */
                        for(i = 0u; i < EZI2C_RW_SIZE; i++)
                        {
                            i2cMasterData.i2cMasterBuffer[i] -= (i + 1);
                        }
                        
                        UART_PutString("The data is modified by the master");
                        
                        for(i = 0u; i < EZI2C_RW_SIZE; i++)
                        {
                            char message[50];
                            sprintf(message, "Buffer[%d] = %d\r\n", i, i2cMasterData.i2cMasterBuffer[i]);
                            UART_PutString(message);
                        }

                        
                        /* Write data to the EZI2C (RW portion only).
                        *  When writing to EZI2C, write the offset byte followed by the write data. */
                        status = I2C_MasterWriteToEZI2C(EZI2C_SLAVE_ADDR, 1 + EZI2C_RW_SIZE);
                    }
                }
                else
                {
                    /* translate from I2CM_MasterReadBuf() error output to
                    *  I2CM_MasterStatus() error output */
                    status = I2CM_MSTAT_ERR_XFER;
                    
                    UART_PutString("Error");
                }
            }
        }
        
        CyDelay(1000);
            
    }
    
    
        
} /* end of main loop */
 /* end of main() */



uint8 I2C_MasterWriteToEZI2C(uint8 i2CAddr, uint8 nbytes)
{
    uint8 volatile status;
    
    status = I2CM_MasterClearStatus();
    if(!(status & I2CM_MSTAT_ERR_XFER))
    {
        status = I2CM_MasterWriteBuf(i2CAddr, (uint8 *)&i2cMasterData, nbytes,
                                     I2CM_MODE_COMPLETE_XFER);
        if(status == I2CM_MSTR_NO_ERROR)
        {
            /* wait for write complete and no error */
            do
            {
                status = I2CM_MasterStatus();
            } while((status & (I2CM_MSTAT_WR_CMPLT | I2CM_MSTAT_ERR_XFER)) == 0u);
        }
        else
        {
            /* translate from I2CM_MasterWriteBuf() error output to
            *  I2CM_MasterStatus() error output */
            status = I2CM_MSTAT_ERR_XFER;
        }
    }
    
    return status;
}

/* [] END OF FILE */
```



In the above code we are setting up a buffer in the slave.
