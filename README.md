# Test Setup
We need to make the DUT run a test routine (or test pattern), read the results back, and check that they are correct. this involves a couple of steps.
#### Step Zero: Generate test pattern
***(TBD)***
#### Step One: Send the pattern to the DUT

Send the pattern to the DUT so it performs the procedure we want to test. We will do this by loading a peripheral EEPROM with the instructions and ***(data/registers)*** that represent the pattern. On boot, the EEPROM will connect to the DUT and send it the instructions ***(via JTAG for example).*** When the ***(JTAG)*** operations are all complete, we will know the DUT can boot properly.
***(In this phase, the system clock is connected to the EEPROM for sync, and to the DUT TCK line, while it is in JTAG debug mode.)***

So, let's recap:
1. Decide on the test pattern and convert it to instructions ***(and data/registers)*** for the DUT
2. Connect to the peripheral EEPROM and send it the instructions ***(via some method)***
3. Connect the EEPROM to DUT ***(probably via JTAG)***
4. Send the EEPROM contents to the DUT
5. When the send is successful, we can power up and boot the DUT

#### Step Two: Run the DUT until it completes all instructions
