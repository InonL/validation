# Test Setup
This document describes an automated validation test setup for a chip (the DUT) with an embedded processor in it. The test involves sending a routine (or test pattern) to the DUT using a host PC, running the test on the DUT in real time while measuring performance, reading the results back from the DUT, and finally parsing and analyzing the results. 

The chip is connected to a PCB which provides all the necessary I/O:
- Voltage 
- Clock input
- Power-on reset (held low at power-up to delay boot until explicit instruction)
- Serial interface
- Peripheral EEPROM (connected via SPI) to load the test instructions (since the chip doesn't have internal memory)

In addition, the following test equipment is used:
- Voltage meter - used to monitor the DUT supply line for voltage spikes or changes which can affect the chip's performance, and also for measuring DUT power in combination with the current meter
- Current meter - used to measure max, average, and idle current consumption of the DUT, and to measure DUT power together with the voltage meter
- Oscilloscope - for monitoring the rising or falling edges of critical signals
- Thermometer - measures temperature on the DUT package when idle and during operation
- Controlled DC power supply - used to switch the test board on and off in a controlled way

The test setup is managed by a host PC. It communicates with the PCB and all test equipment using serial connections and controls them with a Python environment that runs the test procedure.
### Setup Diagram

![](setup_schematic.svg)
The PC uses serial connections to communicate with the test equipment and the test PCB. The UART connection on the PCB is connected to a UART Controller on the board, which decodes the messages sent from the host PC and routes them to different components on the board using a demultiplexer.
### Python Test Environment
The Host PC runs the test and controls the test equipment with a Python environment.
The environment includes the following elements:
- Test pattern generator - generates random operands, converts them to binary instructions for loading onto the EEPROM, and stores them in a file. It also generates reference results to compare with the actual test results
- SerialDevice class - base class for all serial devices, which takes serial connection parameters such as baud rate, COM port, etc.
- Specific serial device classes - every unique device class inherits from the base SerialDevice class, and implements device-specific methods. For example - The power supply class will implement a method to set voltage/current values, the Oscilloscope class will implement methods to set trigger values, etc.
- TestBoard class - also inherits from SerialDevice, but this class is unique because it's used to communicate with several different components on the test board. It includes methods such as:
	- Component select - When calling this method, a special message is sent so the UART controller on the board knows when to change the selection lines on the demultiplexer according to the desired component
	- JTAG debugging methods - methods used to communicate with DUT via JTAG, including methods to encode and decode JTAG instructions in UART messages
	- SPI methods - similar to the JTAG methods, these are used to encode UART messages to SPI messages compatible with the EEPROM
	- Power-on Reset - a method to toggle the power-on reset component
- Result parser and collector - responsible for parsing communications from the test board and test equipment, such as:
	- JTAG parsing - parses the JTAG debug messages and extracts the relevant information to a file which can later be used for statistical analysis and graph plotting
	- Measurement parsing - parses measurements sent by the various test equipment devices, and orders them in a file for easy viewing
	- Calculation methods - some calculations need to be performed using the test results and measurements, for example: average power consumption, average temperature value, etc.

The following diagram describes the control flow of the test procedure, controlled by the PC using the Python environment:

![](control_flow.svg "a title")
The general steps of the test procedure are as follows:
1. Generate the test patterns and their reference results, convert them to binary instructions, and save to file for sending later
2. Power up the test equipment and check that it communicates properly with the host PC - if not, check for connection errors and repeat
3. Power up of the test board, test that it also communicates properly, and wait for a few seconds in order to measure power, voltage, and temperature while the DUT is idle
4. Select the EEPROM component for communication, change it to write mode, and one by one, send each instruction to it from the file previously created, until done
5. While changing the EEPROM to read mode, select the power-on circuit for communication, and disable it so the DUT will start the boot process
6. Select the DUT JTAG bus for communication, and wait until the DUT sends a message that it is ready to run
7. Send start command to the DUT, while logging the start time, the current program counter value, and start measuring voltage, current, power, and temp values. Also, count number of clock cycles
8. When result is received from DUT, append the result, measurements and counted clock cycles to a file
9. Repeat steps 7 to 8 until the 'Done' flag is raised by the DUT
10. When 'Done' is raised, the test is over
### Parsing and analyzing results
After receiving the test results from the DUT, they will be parsed and collected in a clear way which allows statistical analysis and plotting graphs.
The test environment will save the results into a convenient file format (csv, xls, and other similar formats). Every row will include several columns relevant to the results - program counter value, the instruction operation, input registers and their values, output registers and their values, their reference values (created at the beginning), and comparison results. For example:

| PC         | operation | reg 1 | reg 1 value | reg 2 | reg 2 value | output reg | output value | reference  | PASS/FAIL |
| ---------- | --------- | ----- | ----------- | ----- | ----------- | ---------- | ------------ | ---------- | --------- |
| 0x00000000 | ADD       | 1     | 0x00000006  | 2     | 0x00000005  | 3          | 0x0000000b   | 0x0000000b | PASS      |
| ...        | ...       | ...   | ...         | ...   | ...         | ...        | ...          | ...        | ...       |

Performance measurements files will also be generated using the info collected from the test equipment. For example, a timing measurements file:

| PC         | clock frequency \[Hz] | execution time \[nsec] | clock cycles |
| ---------- | --------------------- | ---------------------- | ------------ |
| 0x00000000 | 10^9                  | 3                      | 3            |
| ...        | ...                   | ...                    | ...          |

An electrical measurements file:

| PC         | min/max/avg V<sub>DD</sub> \[V] | min/max/avg current \[mA] | avg power (W) |
| ---------- | ------------------------------- | ------------------------- | ------------- |
| 0x00000000 | 1.25/1.33/1.29                  | 7200/7700/7450            | 9.75          |
| ...        | ...                             | ...                       | ...           |

And a temperature measurements file:

| PC         | start temp \[<sup>o</sup>C] | end temp \[<sup>o</sup>C] | temp difference \[<sup>o</sup>C] |
| ---------- | --------------------------- | ------------------------- | -------------------------------- |
| 0x00000000 | 45                          | 50                        | 5                                |
| ...        | ...                         | ...                       | ...                              |

Now after the results are ordered in files, they can be used to pinpoint errors in the test, such as:
- Functional errors - test results that do not match the expected (reference) result. Several errors of the same operation or with the same result might indicate a design or manufacturing flaw
- Timing errors - if an instruction executed too fast/slow, the cause can be investigated by looking at the results and other measurements
- Electrical faults - acute changes in voltage/current/power can explain functional and/or timing errors
- Thermal errors - if the package temperatures are too high, the DUT might lower its frequency, operating voltage, etc. This can explain timing errors and high voltage/current/power fluctuations

Using these files, Python libraries such as `numpy` and `matplotlib` can perform statistical analysis of the results and plot graphs.