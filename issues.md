#### No ROM in the chip
Need to ask what that means (can think of three possible options):
* Instructions memory (most probable)
- Data Memory (probable)
* Register memory (least probable)
#### Power-on-Reset
Where does that come in? Before DUT boots or maybe after? When the whole PCB is turned on?
#### Can I measure execution time of instructions and/or patterns?
It seems probable that certain instructions' execution time needs to be checked in order to gauge performance... Need to ask about this
#### In general - load each instruction individually and gauge performance during runtime, or load whole patterns and evaluate them post run?
It's not clear to me what should be the correct way to do this. I can load lots of instructions to an EEPROM, make it send them to the DUT, let it run, and then analyze the results. On the other hand, I can send each instruction individually and then analyze them while other instructions are still running (using the EEPROM as a sort of buffer, or not at all and send directly to the DUT via communication channel).

#### What to focus on - Physical validation or functional?
I am not sure if I should include physical measurements such as power, temperature, execution time, etc.