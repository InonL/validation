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
	- JTAG debugging methods - methods used to communicate with DUT via JTAG, including methods to encode and decode JTAG instructions in UART messages
	- SPI methods - similar to the JTAG methods, a method is needed to encode UART messages to SPI messages compatible with the EEPROM
	- Power-on Reset - a method to toggle the power-on reset component
- Result parser and collector - Needs to be able to parse the JTAG debug messages, collect them, and order them correctly in a file which can later be used for statistical analysis and graph plotting

### Test Procedure Control flow
![](control_flow.svg)
##### Collecting data from DUT
The processor will communicate the results via the JTAG debug connection back to the host PC. Some ways that this can be done:
- Sending info during runtime after every instruction is done - the processor will send debug info at runtime for every instruction via JTAG, for example: Program counter values, register values before and after the instruction, data memory values, etc. This might be hard to perform due to timing issues and data rate limitations on the JTAG and/or UART busses
- Dumping the registers and data memory after all instructions have been completed - this means the generated test patterns will all be finished and their results will be saved into memory. It is harder to measure single instruction execution time in this method, but easier to parse

Further elaboration on the test procedure can be found on the 'procedure.md' file.
### Parsing the results
After receiving the test results from the DUT, they will be parsed and collected in a clear way which allows statistical analysis and plotting graphs.
The test environment will have a way to save the results into a convenient file format (I.e csv, xls, etc.). Every row will include several columns relevant to the results - program counter value, the instruction operation, input registers and their values, output registers and their values, their reference values (created at the beginning), and comparison results. For example:

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

| PC         | avg V<sub>DD</sub> \[V] | V<sub>DD</sub> variance \[V] | avg current \[mA] | current variance \[mA] | avg power (W) |
| ---------- | ----------------------- | ---------------------------- | ----------------- | ---------------------- | ------------- |
| 0x00000000 | 1.3                     | 0.1                          | 7500              | 500                    | 9.75          |
| ...        | ...                     | ...                          | ...               | ...                    | ...           |

And a temperature measurements file (NOT FINAL):

| PC         | start temp \[<sup>o</sup>C] | end temp \[<sup>o</sup>C] | avg temp \[<sup>o</sup>C] |
| ---------- | --------------------------- | ------------------------- | ------------------------- |
| 0x00000000 |                             |                           |                           |
| ...        | ...                         | ...                       | ...                       |

Now after the results are ordered in files, we can use them to pinpoint errors in the test, such as:
- Functional errors - we can now see when the test result does not match the expected (reference) one. Several errors of the same type might indicate a design or manufacturing flaw
- Timing errors - if an instruction executed too fast/slow, we can investigate the cause by looking at the results and other measurements to determine the cause
- Electrical faults - acute changes in voltage/current/power can explain functional and/or timing errors
- Thermal errors - if the package temperatures are too high, the DUT might lower its frequency, operating voltage, etc. This can explain timing errors and high voltage/current/power fluctuations

With these files we can now use Python libraries such as `numpy` and `matplotlib` to perform statistical analysis of the results and plot graphs.