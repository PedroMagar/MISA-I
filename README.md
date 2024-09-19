
# MISA-I
**My ISA version 1** is a 8-bit MISC (stretching really hard its meaning) ISA made to be functional, efficient and competitive against well established architectures (AKA z80 / MOS 6502). The goal is to see how much could be improved upon the aged ISA's if they were designed with a modern concept in mind.
>The specification is still in development and there is still a long way to go...

## Architecture
The architecture will consist of 16 registers are equally divided into 4 registers bank while two banks are active, this makes 8 registers addressable at a time, it' possible to switch which register bank are active through SRB operation. Since 8-bit instruction is too little to hold two full addresses for an 8-register architecture without compromising instruction size, the instruction will specify the Rd (register destiny) while Rs (register source) will be relative to it (when F=0 address will be Rs=Rd+4 while F=1 will be Rs=Rd+3).

### Characteristics:
- Two addresses architecture.
- One Memory Address (MEM_ADDR) register.
- One Program Counter (PC) register.
- 8 Management Registers.
- 16 registers: split in sets of 4 (Register Bank), the instruction field in SRB (Swap Register Bank) will determine which set is to be used.
- SRB will match the register bank accordingly:
	- 00 / LL: Will use the default register bank, making avaliable: r7-r4 and r3-r0.
	- 01 / LH: Will swap the lower register bank, making avaliable:  r7-r4 and r11-r8.
	- 10 / HL: Will swap the higher register bank, making avaliable: r15-r12 and r3-r0.
	- 11 / HH: Will swap both registers banks, making avaliable: r15-r12 and r11-r8.
- 4 Operations Mode:
	- 00: Management - r11-r4 will hold the CPU configuration.
	- 01: 8 bit Low mode - the CPU will use only the 8 less significant bits from its register to do operations.
	- 10: 8 bit High mode - the CPU will use only the 8 most significant bits from its register to do operations.
	- 11: 16 bit mode - the CPU will use the full register size and manipulate data in 16 bit.
- Management:
	- r11: Interruption Address.
	- r10: Interruption timer (cycles).
	- r9: Relative Address.
	- r8: Field Length.
	- r7: CPUID - Current CPU capabilities.
	- r6: NPC - New Core Program Counter
	- r5: FMA - Flex Memory Address.
	- r4: INTM - Interruption message.
	> Currently, the focus of 'Management' mode is to access some CPU information such as what is its capabilities or how is its current operation state, there is also an attempt to implement protected memory, this can be seen from r9 through r8.
 	> CPU Mode (Define how the CPU will operate) is under consideration to come back on r6 or r5.
 	> With the addition of ADDC instruction, the existence of a register to hold LOC (Last Operation Carry) became unclear, with the necessity to allow application to act to the interruption, it became possible to change r4 to allow more flexibility in the architecture.
- CPUID:
  
|Vector |Multiple execution |Flex Memory | Write Police |Protected Memory |Operations (8/16/32-bit) |
|-------|-------------------|------------|--------------|-----------------|-------------------------|
|2-bit  |1-bit              |1-bit       |1-bit         |1-bit            |2-bit                    |

- CPU Mode (Currently removed) - It will be possible to change some CPU behavior, to do that each bit in 'CPU Mode' register will define it:
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
		- 1: CPU will only invalidate instruction cache, effectively working as FENCE.I.
		- 0: FENCE instruction will invalidate instruction cache and retire data cache when called.
	- Vector: (default 0)
		- 1: CPU will enter in vector mode.
		- 0: CPU will operate normally.
- Vector Mode:
	- Will replace the upper 4 registers with vector registers (size to be informed in CPUID)
	- RRR F --> VV RR: Where 'RR' is the operator and will be any of the upper register files (r7-r4 or r15-r12) while 'VV' will be the vector register file which ranges from v3 through v0.
	- RRR ---> RV F: Where 'F' will define if operation is on register file (0) or on the vector registers (1), 'RV' will hold the address of register that the operation must be over.
	- Limitations:
		- AND, OR and XOR will only work over the vector registers.
		- SMEM, APC and JAL will stay unchanged and operate on the original register name, this way it's possible to keep the control flow working without consuming addresses available to vector operations.
		> Branch (BEQz/BGEz) is under review as a valid operation over the vector register file.
- Immediate: Value read from instruction memory.
- RRR F: 
	- RRR: holds the operand register address, all changes will be applied to it.
	- F = 0: This will make the operator have the most significant bit of its address been the inverse the operand
	- F = 1: This will make the operator be the register before the operand (operand_addr -1)
	- If the instruction does not have a function bit 'F', it will operate solely on operand or assume F=0.
- Sleep will stop on an interruption signal, this can be achieved by an interruption pin or interruption timer.
- Flex Memory: A small amount (planned 64/128/256 Bytes) of memory that can be used as a cache or as independent memory.
- FMA: Allows the positioning of the flex memory address, any memory that exceeds the addressable limit can be used as a cache.
- Split Core is a wild concept where the CPU will start operating 2 PC and executing two flows, the idea is if the CPU has Flex Memory, it could execute a sub-program entirely in it's Flex Memory while keeping working on the main program in the primary Thread.
- RST: Will be used as an escape command to run the CPU in a new architecture (something like "CPU will reset if tryed to run unsupported mode"), but also can be used as a "memory flag" to now if the content in the region can be used RER/RERO.

## Instructions
The following table lists the architecture current instructions.

|Binary    |Instruction |Description                             |
|----------|------------|----------------------------------------|
|000 0 0000|NOP         |                                        |
|RRR 0 0001|CLR         |Clear                                   |
|RRR 1 0001|INV         |Inverse                                 |
|RRR F 1001|AND         |                                        |
|RRR F 0101|OR          |                                        |
|RRR F 1101|XOR         |                                        |
|RRR F 0011|SHL         |Shift Left                              |
|RRR F 1011|SHR         |Shift Right                             |
|RRR F 0111|BEQz        |Branch Equal Zero                       |
|RRR F 1111|BGEz        |Branch Greater Equal Zero               |
|RRR F 0010|ADD         |                                        |
|RRR F 1010|SUB         |                                        |
|RRR 1 0110|INC         |Increment                               |
|RRR 0 0110|ADDC        |ADD Carry                               |
|RRR 1 1110|DEC         |Decrement                               |
|RRR 0 1110|LI          |Load Immediate from instruction         |
|RRR 0 0100|LW          |Load Word                               |
|RRR 1 0100|SW          |Store Word                              |
|RRR 0 1100|APC         |Add to Program Counter                  |
|RRR 1 1100|JAL         |Jump and Link                           |
|FFF 0 1000|OPM         |Operation Mode                          |
|RRR 1 1000|SMEM        |Swap Memory Address                     |
|FF0 1 0000|SRB         |Swap Register Bank                      |
|001 1 0000|CSM         |Control Sequential Memory               |
|011 1 0000|CI          |Control Interruption                    |
|101 1 0000|CV          |Control Vector                          |
|111 1 0000|CRS         |Control Rotation and Signal             |
|001 0 0000|SNDI        |Send Interrupt                          |
|011 0 0000|FENCE       |Invalidates I.cache and retires D.cache |
|101 0 0000|SPC         |Split Core                              |
|111 0 0000|SLP         |Sleep (Kill core)                       |
|010 0 0000|RER         |Restore all Registers from memory       |
|110 0 0000|RERO        |Restore Operation Mode Registers        |
|100 0 0000|RST         |Reset                                   |

### Development:
Currently there is a feeling of missing some functionalities, like:
- Become a true MISC architecture: Currently the instruction structure are much more RISC like, and some instructions can even been said to be CISC, here are the main culprits:
	- LI: will read the next instruction (8-bit data mode) or even next two instructions (16-bit data mode) as immediate value to store on register, it's not complex, but can be argued to have a variable length instruction size, even so this instruction is here to stay, cause as 8-bit do not let left much space for an immediate, without it generating an address to read or store a value would became painfully taxing on the runtime.
	- Changing CPU behavior is something to be debated, as if already existing instructions will have a different behavior some may count then as new instructions, or as a complex instruction, or both, this would lead to a violation of philosophy, none less this one is also here to stay, the opcode saved and flexibility added is what gives hope of this ISA been competitive.
	- RER & RERO: They are great for preemptive multitasking, based on XJ from CDC 7600, this function would work perfectly to store the current task when an interrupt is detected or when the OS is doing a context switch, as the architecture doesn't have a JI (Jump Immediate) instruction, it would be impossible to restore all the register when returning to the process because the last register to be restored would have to contain the PC address of when the execution was stopped, this would imply to define in at high level a "throw away" register that would only be usable when interruption is set off.
	- Protected Memory: MISC abolishes MMU, even so I believe it's important to have a way to protect itself from malicious users. I'm still studying how to do it, for now I'm looking at RA (Relative Address) and FL (Field Length) of CDC Architecture.
	- SOLUTIONS: RER/RERO, Protected Memory, Fence and others non essentials behavior probably will be defined as non-obligatory or even postergated to MISA-II, in this way, MISA-I can be kept clean and MISC conformant if desirable (usability will take priority over philosophy).
- Watchdog Timer: Due to instruction space limitations, it was removed in favor of the FENCE instruction, it's currently in the high priority list to be added (currently loking into replace RERO).
- CP/MV: Currently the operation to Copy was removed to resolve a major oversight of not having the 'Inverse' operation, as a bonus the architecture also received an add carry, but the downside is as the architecture does not have a way to directly access all register, when the data needs to interact with two register that doesn't have a link it has to be moved/copied, this can occur in a major overhead as currently the only way to copy is to first clear a register then add the value to be moved/copied to it, and also the data in the other register may need to be moved or even worse, stored. This may look like a design flaw but it's an architectural decision (won't say a good one), is not all data that needs to communicate with every register and current design allow enough flexibility to be reasonably efficient.
- MEMCPY: With the 'Flex Memory' concept, it would help to have an instruction to copy directly the memory without recurring to LW/SW loop.
- BIT Operations: Reading (for a branch) and writing a single bit of a register (flag).
- Flags: A dedicated register for flags and used by bit operations.
- JAL: Jump and Link could be not destructive, in this case it would be possible to swap with SMEM and change SMEM behavior to be based on register file location.
- Cache and TSO (Total Store Order): Let's start simple, 128 Bytes of storage on a 6T Flip-Flop would result in 6144 Transistors, a MOS 6502 has 3500, Intel 8080 has 6000 and Z80 has 8500 ... For an 8-bit CPU, a cache is useless or at least on the best of the best a really really really HUGE burden, BUT, if low transistor count (small size) is not one of the requirements, and a wide input (like 4-way decode) is what you are doing, a small cache can be desired, and when there is a cache, there is a possibility of memory mismatch with other devices on the bus, to prevent this, sync could be done by setting the CPU in write through mode and using FENCE instruction in sensible areas.
- Send Interruption: Interruption can be interpreted as a one bit signal, it would likely violate the RISC/MISC philosophy of only dealing with memory, but its benefits of working as a mean for multi-core synchronization, bank switch and others tasks cannot be underestimated (even if the higher bits of the address space can be used similarly), it could be added by replacing SMEM H (when F=1), this would also free some space for more instructions.

There are also some operations that have some high overhead due to the architecture:
- Branch between values may require a SUB or ADD operation before comparison, due to the destructive nature of the architecture, if the original value of comparison cannot be lost there will be an operation overhead to copy the main value to an available register before the Branch, if there is no available register at the moment... Well... glhf...
- Carry behavior can not be changed directly like others control instructions (CSM, CS and CV), for changing this behavior will be necessary to enter management mode (OPM 0) and then updating 'CPU Mode' register with the desired value/behavior.

### Honorable Mentions
There are some instructions that was once in the isa, but currently were removed due to conflict/priority compared to the current design, but there are also others that are just waiting to be in:

|Binary    |Instruction |Description                             |Candidate|
|----------|------------|----------------------------------------|---------|
|111 0 0000|CWDT        |Clear Watchdog Time                     |Yes      |
|RRR 0 0001|CP          |Copy                                    |Yes      |
|RRR 0 0001|MV          |Move                                    |Yes      |
|RRR 0 0001|MEMCPY      |Memory Copy                             |Yes      |
|RRR I IIII|RNG         |Generate Random Value                   |Yes      |
|III I IIII|MKV         |Make Vector                             |Yes      |
|110 0 0000|BKPR        |Restore Backup from memory              |         |
|110 0 0000|FENCE       |Invalidates I.cache and retires D.cache |         |
|110 0 0000|FENCE.I     |Invalidates instruction cache           |         |
|110 0 0000|FENCE.D     |Retires data cache                      |         |
|RRR 0 1110|FILL        |Fill register with 1                    |         |
|RRR I IIII|BC          |Branch if carry                         |         |
|RRR R IIII|BITW        |Write Bit                               |         |
|III I IIII|BITT        |Test Bit                                |         |
|III I IIII|RPC         |Read PC                                 |         |
|III I IIII|RI          |Read Interruption                       |         |
|III I IIII|CR          |Control Rotation                        |         |
|III I IIII|CC          |Control Carry                           |Yes      |
|III I IIII|CIO         |Control Interruption on Overflow        |Yes      |
|III I IIII|CPUID       |Read CPU capabilities                   |         |

### Code Example
The following is a code example for a multiplication loop in software: 
```
parameters: | accumulator = r1 | base_1 = r2  | base_comp = r3  | done = r4 |
          : | loop = r5        | operand = r6 | operator = r7   |           |
loop:
BEQz 0 operator                 # Checks if there are no more operator to multiply
JP   done                       # Jump to done
CLR  base_comp                  # Clear comparator
INC  base_comp                  # Sets 1 to comparator
AND  base_comp, operator        # Filter operator to see if least significant bit is 0 or 1
DEC  base_comp			# If 1, it will became zero to make branch easy
BEQz base_comp                  # Verify se operator least significant bit is 1
ADD  accumulator, operand            # If true, adds the current value of the operand to the result
SHL  operand, base_1            # Shifts the multiplicand to the left
SHR  operator, base_1           # Shifts the multiplier to the right
jp   loop                       # Repeat the loop
done:
# Result is in r7
```

For reference, a RISC-V 32IC would look like:
```
loop:
    andi t1, x2, 1     # Checks if the least significant bit is 1
    beq  t1, x0, skip  # If not, skip adding
    add  x3, x3, x1    # Adds the current value of the multiplicand to the accumulator
skip:
    slli x1, x1, 1     # Shifts the multiplicand to the left
    srli x2, x2, 1     # Shifts the multiplier to the right
    bnez x2, loop      # If x2 is not zero, repeat the loop
done:
    # Result is in x3
```

Even though RISC-V has less instruction, it's using 16-bit instruction, so it would translate to 12-Bytes of storage, while on MISA-I,  because of its 8-bit instructions, is 'only' using 11-Bytes... Of course, this pseudo advantage can be denied if taken in consideration that as MISA uses more instructions for preparation (clearing registers and setting values), and also the major performance efficiency advantage that RISC-V does have as its needs to process less instruction making its clock more effective (even a MISA with instruction fusion would have its limitations) is hard to compete. Anyway, a win is a win, let's take anything that is possible! In a multiplication loop, MISA-I is 1-byte shorter!
