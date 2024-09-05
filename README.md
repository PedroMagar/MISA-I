# MISA-I
**My ISA version 1** is a 8-bit MISC (stretching really hard its meaning) ISA made to be functional, efficient and competitive against well established architectures (AKA z80 / MOS 6502). The goal is to see how much could be improved upon the aged ISA's if they were designed with a modern concept in mind.
>The specification is still in development and there is still a long way to go...

## Architecture
The architecture will consist of 16 registers while 8 are addressable at a time. Since 8-bit instruction is too little to hold two full addresses for an 8-register architecture without compromising instruction size, the instruction will specify the Rd (register destiny) while Rs (register source) will be relative to it, this switch will be made through SRB operation.
Characteristics:
- Two addresses architecture.
- One register for the path (address register).
- 16 registers: split in sets of 4, the instruction field in SRB will determine which set is to be used.
- SRB will match the register bank accordingly:
	- 00: Will use the default register bank, r7-r4 and r3-r0.
	- 01: Will swap the lower register bank,  r7-r4 and r11-r8.
	- 10: Will swap the higher register bank, r15-r12 and r3-r0.
	- 11: Will swap both registers banks, r15-r12 and r11-r8.
- 4 Operations Mode:
	- 00: Management - r11-r4 will hold the CPU configuration.
	- 01: 8 bit Low mode - the CPU will use only the 8 less significant bits from its register to do operations.
	- 10: 8 bit High mode - the CPU will use only the 8 most significant bits from its register to do operations.
	- 11: 16 bit mode - the CPU will use the full register size and manipulate data in 16 bit.
- Management:
	- r15: Supervisor Address higher bits.
	- r14: Supervisor Address lower bits.
	- r13: Supervisor Interruption Address higher bits.
	- r12: Supervisor Interruption Address lower bits.
	- r11: Reserved Memory Size.
	- r10: Reserved Memory Address.
	- r9: Supervisor Interruption timer (cycles).
	- r7: CPU Info - Current CPU capabilities.
	- r6: CPU Mode - Will define how the CPU is currently operating.
	- r5: Interruption Address higher bits.
	- r4: Interruption Address lower bits.
	- r3: Sleep time (cycles).
	> Currently, the focus of 'Management' mode is to access some CPU information such as what is its capabilities or how is its current operation state, there is also an attempt to implement protected memory, this can be seen from r15 through r9.
- CPU Mode - It will be possible to change some CPU behavior, to do that each bit in 'CPU Mode' register will define it:
	- Interruption: (default 1)
		- 1: CPU will jump to interruption Address as soon as it receives an interrupt signal.
		- 0: CPU will ignore interruption signal.
	- Signaled: (default 1)
		- 1: CPU will execute ADD/SUB considering signals.
		- 0: CPU  will execute ADD/SUB as unassigned operations.
	- Rotation: (default 0)
		- 1: CPU will execute shift operations (SL/SR) with rotation of its bits.
		- 0: CPU will execute shift operations (SL/SR) without rotation of its bits, filling with 0.
	- Carry: (default 1)
		- 1: CPU will include the Carry bit of the last ADD/SUB operation in a multi ADD/SUB sequence.
		- 0: CPU will not include the Carry bit in any ADD/SUB operation.
	- Sequential Memory: (default 0)
		- 1: CPU will update memory address after a memory operation (Read/Write) with +1.
		- 0: CPU will not update memory address after a memory operation (Read/Write).
	- Write-Through: (default 1)
		- 1: If the CPU has a cache, it will always write changes to memory.
		- 0: If the CPU has a cache, it will only write changes to memory on cache line retirement.
	- FENCE: (default 0)
		- 1: CPU will only invalidates instruction cache, effectively working as FENCE.I.
		- 0: FENCE instruction will invalidate instruction cache and retire data cache when called.
	- Vector: (default 0)
		- 1: CPU will enter in vector mode.
		- 0: CPU will operate normally.
- Vector Mode:
	- Will replace the upper 4 registers with vector registers (size to be informed in CPUID)
	- RRR F --> VV RR: Where 'RR' is the operator and will be any of the upper register file (r7-r4 or r15-r12) while 'VV' will be the vector register file which ranges from v3 through v0.
	- RRR ---> RV F: Where 'F' will define if operation is on register file (0) or on the vector registers (1), 'RV' will hold the address of register that the operation must be over.
	- Limitations:
		- AND, OR and XOR will only work over vector registers.
		- SMEM, APC and JAL will stay unchanged and operate on the original register name, this way it's possible to keep the control flow working without consuming addresses available to vector operations.
		> Branch (BEQz/BGEz) is under review as a valid operation over vector register file.
- Immediate: Value read from instruction memory.
- RRR F: 
	- RRR: holds the operand register address, all changes will be applied to it.
	- F = 0: This will make the operator have the most significant bit of its address been the inverse the operand
	- F = 1: This will make the operator be the register before the operand (operand_addr -1)
	- If the instruction does not have a function bit 'F', it will operate solely on operand or assume F=0.
- Sleep will start as soon as a value is written on the 'Sleep' register.
- RST: Can be used as an escape command to run the CPU in a new architecture.

## Instructions
The following table lists the architecture current instructions.

|Binary    |Instruction |Description                             |
|----------|------------|----------------------------------------|
|000 0 0000|NOP         |                                        |
|RRR F 0001|CP          |Copy                                    |
|RRR F 1001|AND         |                                        |
|RRR F 0101|OR          |                                        |
|RRR F 1101|XOR         |                                        |
|RRR F 0011|SL          |Shift Left                              |
|RRR F 1011|SR          |Shift Right                             |
|RRR F 0111|BEQz        |Branch Equal Zero                       |
|RRR F 1111|BGEz        |Branch Greater Equal Zero               |
|RRR F 0010|ADD         |                                        |
|RRR F 1010|SUB         |                                        |
|RRR 1 0110|INC         |Increment                               |
|RRR 0 0110|CLR         |Clear                                   |
|RRR 1 1110|DEC         |Decrement                               |
|RRR 0 1110|LI          |Load Immediate from instruction         |
|RRR 0 0100|LW          |Load Word                               |
|RRR 1 0100|SW          |Store Word                              |
|RRR 0 1100|APC         |Add to Program Counter                  |
|RRR 1 1100|JAL         |Jump and Link                           |
|RRR 0 1000|            |**TO-DO**                               |
|RRR 1 1000|SMEM        |Swap Memory Address                     |
|II0 1 0000|SRB         |Swap Register Bank                      |
|II1 1 0000|OPM         |Operation Mode                          |
|001 0 0000|CSM         |Control Sequential Memory               |
|011 0 0000|CS          |Control Signaled                        |
|101 0 0000|CV          |Control Vector                          |
|111 0 0000|FENCE       |Invalidates I.cache and retires D.cache |
|010 0 0000|BKP         |Backup Registers to memory              |
|110 0 0000|BKPR        |Restore Backup from memory              |
|100 0 0000|RST         |Reset                                   |

### Development: Current missing
Currently there is a feeling of missing some functionalities, like:
- Become a true MISC architecture: Currently the instruction structure are much more RISC like, and some instructions can even been said to be CISC, changing CPU behavior is something to be debated as if already existing instructions will count as new one, here are the main corrupt:
	- LI: will read the next instruction (8-bit data mode) or even next two instructions (16-bit data mode) as immediate value to store on register, it's not complex, but can be argued to have a variable length instruction size, even so this instruction is here to stay, cause as 8-bit do not let left much space for an immediate, without it generating an address to read or store a value would became painfully taxing on the runtime.
	- BKP & BKPR: They are great for preemptive multitasking, based on XJ from CDC 7600, this function would work perfectly to store the current task when an interrupt is detected or when the OS is doing a context switch, as the architecture doesn't have a JI (Jump Immediate) instruction, it would be impossible to restore all the register when returning to the process as the last register to be restored would have to contain the PC address of when the execution was exchanged, this would imply to define in a high level a "throw away" register that would only be used with interrupt off.
	- Protected Memory: MISC abolishes MMU, even so I believe it's important to have a way to protect itself from malicious users. I'm still studying how to do it, for now I'm looking at RA (Relative Address) and FL (Field Length) of CDC Architecture.
	- SOLUTIONS: BKP/BKP, Protected Memory, Fence and others non essentials behavior probably will be defined as non-obligatory or even postergated to MISA-II, in this way MISA-I can be kept clean and MISC conformant if desired (usability will take priority over philosophy).
- Watchdog Timer: Due to instruction space limitations, it was removed in favor of the FENCE instruction, it's currently in the high priority list to be added.
- BIT Operations: Reading (for a branch) and writing a single bit of a register (flag).
- Flags: Can be used to read the processor current operation mode without going to management mode.
- JAL: Jump and Link could be not destructive, in this case it would be possible to swap with SMEM and change SMEM behavior to be based on register file location.
- Cache and TSO (Total Store Order): Let's start simple, 128 Bytes on a 6T Flip-Flop would result in 6144 Transistors, a MOS 6502 had 3500 Transistors... For an 8-bit CPU, a cache is useless or at least on the best of the best a really really really HUGE burden, BUT, if low transistor count (small size) is not one of the requirements, and a wide input (like 4-way decode) is what you are doing, a small cache can be desired, and when there is a cache, there is a possibility of memory mismatch with other devices on the bus, to prevent this sync could be done setting the CPU in write through mode and using FENCE instruction in sensible areas.
- Send Interruption: Interruption can be interpreted as a one bit signal, it would likely violate the RISC/MISC philosophy of only dealing with memory, but its benefits of working as a mean for multi-core synchronization, bank switch and others tasks cannot be underestimated (even if the higher bits of the address space can be used similarly), it could be added by replacing SMEM H (when F=1), this would also liberate some space for more instructions.

There are also some operations that have some high overhead due to the architecture:
- Branch between values may require a SUB or ADD operation before comparison, due to the destructive nature of the architecture, if the original value of comparison cannot be lost there will be an operation overhead to copy the main value to an available register before the Branch, if there is no available register at the moment... Well... glhf...
- Rotation, Carry and Interruption can not be changed directly like others control instructions (CSM, CS and CV), for changing this behavior will be necessary to enter management mode (OPM 0) and then updating 'CPU Mode' register with the desired value/behavior.

### Honorable Mentions
There are some instructions that was once in the isa, but currently were removed due to conflict/priority compared to the current design, but there are also others that are just waiting to be in:

|Binary    |Instruction |Description                             |Candidate|
|----------|------------|----------------------------------------|---------|
|III I IIII|SI          |Send Interrupt                          |Yes      |
|111 0 0000|CWDT        |Clear Watchdog Time                     |Yes      |
|110 0 0000|FENCE       |Invalidates I.cache and retires D.cache |Yes      |
|110 0 0000|FENCE.I     |Invalidates instruction cache           |         |
|110 0 0000|FENCE.D     |Retires data cache                      |         |
|RRR 0 1110|FILL        |Fill register with 1                    |         |
|RRR 0 1110|SWPF        |Swap Flags                              |         |
|III I IIII|SLPI        |Sleep until Interrupt                   |         |
|RRR I IIII|BC          |Branch if carry                         |         |
|RRR R IIII|BITW        |Write Bit                               |         |
|III I IIII|BITT        |Test Bit                                |         |
|III I IIII|MKV         |Make Vector                             |Yes      |
|III I IIII|RPC         |Read PC                                 |         |
|III I IIII|RI          |Read Interruption                       |         |
|RRR I IIII|RNG         |Generate Random Value                   |         |
|III I IIII|CR          |Control Rotation                        |Yes      |
|III I IIII|CC          |Control Carry                           |Yes      |
|011 0 0000|CI          |Control Interruption                    |Yes      |
|III I IIII|CIO         |Control Interruption on Overflow        |Yes      |
|III I IIII|MP          |Make Process                            |         |
|III I IIII|RPR         |Reserve Private Memory                  |         |
|III I IIII|RPU         |Reserve Public Memory                   |         |
|III I IIII|EADD        |Exception Address                       |         |
|III I IIII|CPUID       |Read CPU capabilities                   |Yes      |
