# Test Setup
We need to make the DUT run a test routine (or test pattern), read the results back, check that they are correct, and measure performance metrics such as execution time, power consumption, and temperature values. This involves a couple of steps.
#### Step Zero: Generate test pattern
***(TBD)***
#### Step One: Preparing the Setup
This phase involves connecting all the necessary equipment in order for the test to commence, such as:
- Connecting test equipment to the host PC, for example: Controlled DC Power Supply, Voltage/Current/Power meters, Oscilloscope, Thermometer, etc.
- Connecting all relevant connections to the Test PCB: DC Power, Voltage/Current/Power probes, Thermocouple on the DUT, etc.
- Preparing the host PC - Executing software, checking all equipment is connected and communicating properly
#### Step Two: Powerup of the Test PCB
We can powerup the test board using a **controlled DC Power Supply**. Several things that can happen in this phase are:
- Power-On-Reset applied to the DUT, it does not boot properly yet
- EEPROM set to write mode, PC serial interface routed to it
- Taking turn-on power measurements (power that DUT consumes during powerup)
- Taking reference idle power measurements
- Taking reference temperature measurements
#### Step Three: Send the pattern to the DUT
Send the pattern to the DUT so it performs the procedure we want to test. We will do this by loading a peripheral EEPROM with the instructions and ***(data/registers)*** that represent the pattern. On boot, the EEPROM will connect to the DUT and send it the instructions ***(via JTAG for example).*** When the ***(JTAG)*** operations are all complete, we will know the DUT can boot properly.
***(In this phase, the system clock is connected to the EEPROM for sync, and to the DUT TCK line, while it is in JTAG debug mode.)***

So, let's recap:
1. Decide on the test pattern and convert it to instructions ***(and data/registers)*** for the DUT
2. Connect to the peripheral EEPROM and send it the instructions ***(via some method)***
3. Connect the EEPROM to DUT ***(probably via JTAG)***
4. Send the EEPROM contents to the DUT
5. When the send is successful, we can power up and boot the DUT

#### Step Four: Run the DUT until it completes all instructions
Now, the DUT is ready to run the test. We connect the system clock to the DUT clock line, and ***(reset or run?)*** the DUT. It's memory is now loaded, and so after we wait a known number of clock cycles, ideally, a 'Done' line from the DUT should rise and now we know the test pattern completed. 