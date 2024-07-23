This document elaborates on the test procedure described in the control flow diagram on the readme file.
### Step Zero: Generate test pattern
Before starting the test procedure, a test pattern needs to be provided in the form of a series of instructions for the DUT embedded processor. Different processor instructions need to be tested - ALU operations (i.e ADD, AND, or XOR), memory operations (loading/storing data), and so on.
There are many ways to generate instructions operands - For this test I will focus only on random operands for simplicity.

To generate a random operand, the following function will be used:
~~~py
def gen_rand_operands(min, max):
	return random.randint(min, max), random.randint(min, max)
~~~
The function takes minimum and maximum operand values as arguments, and returns a tuple of two operands for the instruction.

### Step One: Preparing the Setup
This phase involves connecting all the necessary equipment in order for the test to commence, such as:
- Connecting test equipment to the host PC, for example: Controlled DC Power Supply, Voltage/Current/Power meters, Oscilloscope, Thermometer, etc.
- Connecting all relevant connections to the Test PCB: DC Power, Voltage/Current/Power probes, Thermocouple on the DUT, etc.
- Preparing the host PC - Executing software, checking all equipment is connected and communicating properly

We initialize the serial communication channels for each piece of test equipment by creating a Device class for each one, like this:
~~~py
power_supply = Device(COM2, 115200)
~~~
### Step Two: Powerup of the Test PCB
We can powerup the test board using a **controlled DC Power Supply**. Several things that can happen in this phase are:
- Power-On-Reset applied to the DUT, it does not try to boot yet
- EEPROM set to write mode, PC serial interface routed to it
- Taking turn-on power measurements (power that DUT consumes during powerup)
- Taking reference idle power measurements
- Taking reference temperature measurements
### Step Three: Send the pattern to the DUT
Send the pattern to the DUT so it performs the procedure we want to test. We will do this by loading a peripheral EEPROM with the instructions and ***(data/registers)*** that represent the pattern. On boot, the EEPROM will connect to the DUT and send it the instructions ***(via JTAG for example).*** When the ***(JTAG)*** operations are all complete, we will know the DUT can boot properly.
***(In this phase, the system clock is connected to the EEPROM for sync, and to the DUT TCK line, while it is in JTAG debug mode.)***

So, let's recap:
1. Decide on the test pattern and convert it to instructions ***(and data/registers)*** for the DUT
2. Connect to the peripheral EEPROM and send it the instructions ***(via some method)***
3. Connect the EEPROM to DUT ***(probably via JTAG)***
4. Send the EEPROM contents to the DUT
5. When the send is successful, we can power up and boot the DUT

### Step Four: Run the DUT until it completes all instructions
Now, the DUT is ready to run the test. We connect the system clock to the DUT clock line, and ***(reset or run?)*** the DUT. It's memory is now loaded, and so after we wait a known number of clock cycles, ideally, a 'Done' line from the DUT should rise and now we know the test pattern completed. 