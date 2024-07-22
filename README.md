# Test Setup
This document describes an automated validation test setup for a generic chip (the DUT) with an embedded processor in it. The test involves sending a routine (or test pattern) to the DUT using a host PC, running the test on the DUT in real time while measuring performance, reading the results back from the DUT, and finally parsing and analyzing the results. 

The chip is connected to a PCB which provides all the necessary I/O:
- Voltage 
- Clock input
- Power-on reset
- Serial interface
- Peripheral EEPROM (connected via SPI) to load the test instructions (as the chip doesn't have internal memory)

In addition, the following test equipment is used:
- Voltage meter - to watch the DUT supply line for voltage spikes or changes which can affect the chip's performance, and also for measuring DUT power in combination with the current meter
- Current meter - used to measure max, average, and idle current consumption of the DUT, and to measure DUT power in combination with the voltage meter
- Oscilloscope - for watching the rising/falling edge of critical signals
- Thermometer - measures temperature on the DUT package under idle load and while running
- Controlled DC power supply - used to switch the test board on and off in a controlled way

And lastly, the Host PC - It is connected to the PCB and all test equipment using serial connections, and controls them using a Python environment.
### Setup Diagram

![](setup_schematic.svg)
The PC uses serial connections to communicate with the test equipment, and with the test PCB. The UART connection on the PCB is connected to a UART Controller on the board, which decodes the messages sent from the host PC and routes them to different components on the board using a demultiplexer.
### Python Test Environment
The Host PC runs the test and controls the test equipment using a Python code environment.
The environment includes elements such as:
- Test pattern generator - generates random operands, converts them to binary instructions for loading onto the EEPROM, and stores them in a file. Should also generate reference results to compare with the actual test results
- SerialDevice class - base class for all serial devices, which takes serial connection parameters such as baud rate, COM port, etc.
- Specific serial device classes - every unique device class inherits from the base SerialDevice class, and implements device-specific methods. For example - The power supply class will implement a method to set voltage/current values, the Oscilloscope class will implement methods to set trigger values, and so on
- TestBoard class - also inherits from SerialDevice, but this class is unique because it's used to communicate with several different components on the test board. It should have the following methods:
	- Component select - When calling this method, the UART controller on the board changes the selection lines on the demultiplexer according to the desired component
	- JTAG debugging methods - methods used to communicate with DUT via JTAG, including a method to encode JTAG instructions in UART messages, parse JTAG messages received from DUT, and save them to a file in-order
	- SPI methods - similar to the JTAG methods, a method is needed to encode UART messages to SPI messages compatible with the EEPROM
	- Power-on Reset - a method to toggle the power-on reset component

The test procedure control flow is as follows:
![](control_flow.svg)

The following section attempts to outline the test procedure using multiple steps.
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