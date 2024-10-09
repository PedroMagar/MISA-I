
# MISA-I
**My ISA version 1** is a 8-bit MISC ISA (in spirit) made to be functional, efficient and competitive against well established architectures (AKA z80 / MOS 6502). The goal is to see how the aged ISAs would fared if they had to compete with a modern design.
>The specification is still in development and there is still a long way to go...

## Architecture
The architecture will consist of 16 registers are equally divided into 4 registers bank while two banks are active, this makes 8 registers addressable at a time, it' possible to switch which register bank are active through SRB operation. Since 8-bit instruction is too little to hold two full addresses for an 8-register architecture without compromising instruction size, the instruction will specify the Rd (register destiny) while Rs (register source) will be relative to it (when F=0 address will be Rs=Rd+4 while F=1 will be Rs=Rd+3).

### Characteristics:
- One Program Counter (PC) register.
- One Memory Address (MEM_ADDR) register.
- 8 Management Registers.
- 16 registers: split in sets of 4 (Register Bank), the instruction field in SRB (Swap Register Bank) will determine which set is to be used.
- SRB will match the register bank accordingly:
	- 00 / LL: Will use the default register bank, making avaliable: r7-r4 and r3-r0.
	- 01 / LH: Will swap the lower register bank, making avaliable:  r7-r4 and r11-r8.
	- 10 / HL: Will swap the higher register bank, making avaliable: r15-r12 and r3-r0.
	- 11 / HH: Will swap both registers banks, making avaliable: r15-r12 and r11-r8.
- Two addresses architecture. **[Rd = Rd (op) Rs]**
- Instructions that operate with two-registers will always assume Rd = RRR from instruction, while Rs is RRR+4 if function bit (F) is 0 or RRR+3 if function bit (F) is 1.
 > Rs address is still under consideration, could be RRR+4 and RRR+1|RRR-1.
- Carry behavior: Instructions that could exceed registers capacity (ADD, SUB, SHF) will fill an internal bit (carry) that can not be read directly but can be used for flow control (branch) with appropriate instruction (BC).
- Branch Behavior: Skip one instruction if false.
  > Behavior is still under review, maybe could skip two instructions (it would allow a SRB or SMEM instruction before jump), or maybe could utilize another register for address.
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
	> Currently, the focus of 'Management' mode is to access some CPU information such as what is its capabilities or how is its current operation state, there is also an attempt to implement protected memory, this can be seen with r9 and r8, restore instructions (RER) would refuse to restore an "higher privileged" (out of bounds) state, a restore in the max address could also indirectly be used to return control to the caller.
 	> CPU Mode (Define how the CPU will operate) is under consideration to come back on r6, r5 or r4.

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
	- RRR F --> VV RR: Where 'RR' is the operator and will be any of the lower register files (r3-r0 or r11-r8) while 'VV' will be the vectorized reference of the full register file:
		- v0: bank 0 - Registers from r0 to r3.
		- v1: bank 1 - Registers from r4 to r7.
		- v2: bank 2 - Registers from r8 to r11.
		- v3: bank 3 - Registers from r12 to r15.
	- RRR ---> RV F: Where 'F' will define if operation is on register file (0) or on the vector registers (1), 'RV' will hold the address of register or vector that the operation must be over.
		- ex: RV F = ```INC 10 1``` will increment v2 (all registers in register bank 2) by 1, while ```INC 10 0``` will increment the current r2 from active register bank.
	- Limitations:
		- SMEM, APC and JAL will stay unchanged and operate on the original register name, this way it's possible to keep the control flow working without consuming addresses available to vector operations.
		> Branch (BEQz/BNEz) is under review as a valid operation over the vector register file.
- Immediate: Value read from instruction memory.
- RRR F: 
	- RRR: holds the operand register address, all changes will be applied to it.
	- F = 0: This will make the operator be operand + 4 (RRR + 4)
	- F = 1: This will make the operator be operand + 3 (RRR + 3)
	- If the instruction does not have a function bit 'F', it will operate solely on operand or assume F=0.
- Sleep will stop at interruption signal, this can be achieved by an interruption pin or interruption timer.
- Flex Memory: A small amount (planned 64/128/256 Bytes) of memory that can be used as a cache or as independent memory.
- FMA: Allows the positioning of the flex memory address, any memory that exceeds the addressable limit can be used as a cache.
- Split Core is a wild concept where the CPU will start operating 2 PC and executing two flows, the idea is if the CPU has Flex Memory, it could execute a sub-program entirely in it's Flex Memory while keeping working on the main program in the primary Thread.
- RST: Will be used as an escape command to run the CPU in a new architecture (something like "CPU will reset if tryed to run unsupported mode"), but also can be used as a "memory flag" to know if the content in the region can be used RER.

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
|RRR F 0011|SHF         |Shift                                   |
|RRR 0 1011|BEQz        |Branch if Equal to Zero                 |
|RRR 1 1011|BNEz        |Branch if Not Equal to Zero             |
|RRR F 0111|MUL*        |                                        |
|RRR F 1111|DIV*        |                                        |
|RRR F 0010|ADD         |                                        |
|RRR F 1010|SUB         |                                        |
|RRR 1 0110|INC         |Increment                               |
|RRR 0 0110|BC          |Branch if Carry                         |
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
|101 1 0000|CRS         |Control Rotation and Signal             |
|111 1 0000|FENCE*      |Invalidates I.cache and retires D.cache |
|001 0 0000|SDI         |Send Interrupt                          |
|011 0 0000|SVIM*       |Set Vector Interaction Mode             |
|101 0 0000|SMIM*       |Set Multi Interaction Mode              |
|111 0 0000|SSIM*       |Set Single Interaction Mode             |
|010 0 0000|RER*        |Restore all Registers from memory       |
|110 0 0000|SLP         |Sleep (Kill core)                       |
|100 0 0000|RST         |Reset                                   |
> Instructions under review:
> - \* : Not mandatory instructions
> - RST: Why a reset instruction? Its main focus would be to change the CPU operation mode to a new version (16-bit), if you try to change and the CPU does not support, it would simply reset, also could be used as a memory flag for XJ to know that the memory region to be restored is a valid one.
> - FENCE is something good to have, if someone tried a 1000-core cpu, having a way to sync cache can be useful, also a future 16/32-bit evolution that could enter "8-bit compact mode" will have a way to stays true to its cache.
> - BEQ/BGE: MISA initially was designed with BEQ/BGE in mind (that is why the function field was not used), the design could return to it.
> - RERO: High likely to be removed, maybe a new CBN (Control Branch Negate) would be a better use of OP Code.
> - All 'control' instructions could be assigned to new management register designated for it.
> - SPC: 'Split Core' concept can currently be achieved by populating NPC with a value, this feature is highly likely to be removed completely, doing so would free a register to hold 'control flags'.
> - ADDC can be replaced by BC


### Development:
Currently there is a feeling of missing some functionalities, like:
- Become a true MISC architecture: Currently the instruction structure are much more RISC like, and some instructions can even been said to be CISC, here are the main culprits:
	- LI: will read the next instruction (8-bit data mode) or even next two instructions (16-bit data mode) as immediate value to store on register, it's not complex, but can be argued to have a variable length instruction size, even so this instruction is here to stay, cause as 8-bit do not let left much space for an immediate, without it generating an address to read or store a value would became painfully taxing on the runtime.
	- Changing CPU behavior is something to be debated, as if already existing instructions will have a different behavior some may count then as new instructions, or as a complex instruction, or both, this would lead to a violation of philosophy, none less this one is also here to stay, the opcode saved and flexibility added is what gives hope of this ISA be competitive.
	- RER: Problably a CISC instructions, It's great for preemptive multitasking, based on XJ from CDC 7600, this function would work perfectly to store the current task when an interrupt is detected or when the OS is doing a context switch, as the architecture doesn't have a JI (Jump Immediate) instruction, it would be impossible to restore all the register when returning to the process because the last register to be restored would have to contain the PC address of when the execution was stopped, this would imply to define at high level a "throw away" register that would only be usable when interruption is set off.
	- Protected Memory: MISC abolishes MMU, even so I believe it's important to have a way to protect itself from malicious users. I'm still studying how to do it, for now I'm looking at RA (Relative Address) and FL (Field Length) of CDC Architecture.
	- SOLUTIONS: RER/RERO, Protected Memory, Fence and others non essentials behavior probably will be defined as non-obligatory or even postergated to MISA-II, in this way, MISA-I can be kept clean and MISC conformant if desirable (usability will take priority over philosophy).
- Watchdog Timer: Due to instruction space limitations, it was removed in favor of the FENCE instruction, it's currently in the high priority list to be added (currently loking into replace RERO).
- MEMCPY: With the 'Flex Memory' concept, it would help to have an instruction to copy directly the memory without recurring to LW/SW loop.
- BIT Operations: Reading (for a branch) and writing a single bit of a register (flag).
- Flags: A dedicated register for flags and used by bit operations.
- JAL: Jump and Link could be not destructive, in this case it would be possible to swap with SMEM and change SMEM behavior to be based on register file location.
- Cache and TSO (Total Store Order): Let's start simple, 128 Bytes of storage on a 6T Flip-Flop would result in 6144 Transistors, a MOS 6502 has 3500, Intel 8080 has 6000 and Z80 has 8500 ... For an 8-bit CPU, a cache is useless or at least on the best of the best a really really really HUGE burden, BUT, if low transistor count (small size) is not one of the requirements, and a wide input (like 4-way decode) is what you are doing, a small cache can be desired, and when there is a cache, there is a possibility of memory mismatch with other devices on the bus, to prevent this, sync could be done by setting the CPU in write through mode and using FENCE instruction in sensible areas.
- Send Interruption: Interruption can be interpreted as a one bit signal, it would likely violate the RISC/MISC philosophy of only dealing with memory, but its benefits of working as a mean for multi-core synchronization, bank switch and others tasks cannot be underestimated (even if the higher bits of the address space can be used similarly), all this flexibility for the small price of an almost throw away instruction space can not be classified as anything other than a bargain, this is why it's here and probably to stay.

There are also some operations that have some high overhead due to the architecture:
- Branch between values may require a SUB or ADD operation before comparison, due to the destructive nature of the architecture, if the original value of comparison cannot be lost there will be an operation overhead to copy the main value to an available register before the Branch, if there is no available register at the moment... Well... glhf...
- CP/MV: Currently the operation to Copy was removed to resolve a major oversight of not having the 'Inverse' operation, as a bonus the architecture also received a branch if carry operation, the downside is since the architecture does not have a way to directly access all register, when the data needs to interact with two register that doesn't have a link betwen then, the data has to be moved/copied, now this can occur in a major overhead as currently the only way to copy is to first clear a register then add the value to be moved/copied to it, and also the data in the other register may need to be moved or even worse, stored. This may look like a design flaw but it's an architectural decision (won't say a good one), is not all data that needs to communicate with every register and current design allow enough flexibility to be reasonably efficient.

### Honorable Mentions
There are some instructions that was once in the isa, but currently were removed due to conflict/priority compared to the current design, but there are also others that are just waiting to be in:

|Binary    |Instruction |Description                             |Candidate|
|----------|------------|----------------------------------------|---------|
|RRR F 1011|SHR         |Shift Right                             |Yes      |
|RRR 0 0110|ADDC        |ADD Carry                               |         |
|RRR F 0111|BEQ         |Branch if Equal                         |Yes      |
|RRR F 1111|BGE         |Branch if Greater or Equal              |Yes      |
|RRR F 1111|BLT         |Branch if Less Than                     |         |
|RRR F IIII|CBN         |Control Branch Negate                   |         |
|111 0 0000|CWDT        |Clear Watchdog Time                     |Yes      |
|RRR 0 0001|CP          |Copy                                    |Yes      |
|RRR 0 0001|MV          |Move                                    |Yes      |
|RRR 0 0001|MEMCPY      |Memory Copy                             |Yes      |
|RRR I IIII|RNG         |Generate Random Value                   |Yes      |
|110 0 0000|RERO*       |Restore Operation Mode Registers        |         |
|110 0 0000|BKPR        |Restore Backup from memory              |         |
|110 0 0000|FENCE.I     |Invalidates instruction cache           |         |
|110 0 0000|FENCE.D     |Retires data cache                      |         |
|RRR I IIII|BC          |Branch if carry                         |         |
|RRR R IIII|BITW        |Write Bit                               |         |
|III I IIII|BITT        |Test Bit                                |         |
|III I IIII|RPC         |Read PC                                 |         |
|III I IIII|RI          |Read Interruption                       |         |
|III I IIII|CR          |Control Rotation                        |No       |
|III I IIII|CS          |Control Signal                          |No       |
|III I IIII|CC          |Control Carry                           |Yes      |
|III I IIII|CIO         |Control Interruption on Overflow        |Yes      |
|III I IIII|CWD         |Control Watchdog                        |Yes      |
|RRR I IIII|CPUID       |Read CPU capabilities                   |         |

### Code Example
The following is a code example for a multiplication loop in software: 
```
parameters: | done = r0       | loop = r2        | operator = r1 | operand = r3  |
          : | base_comp = r5  | accumulator = r7 | base_1 = r6   | base_n_1 = r4 |
loop:
    CLR  base_comp                  # Clear comparator
    INC  base_comp                  # Sets 1 to comparator
    AND  base_comp, operator        # Filter operator to see if least significant bit is 0 or 1
    BNEz base_comp                  # Verify se operator least significant bit is 1
    ADD  accumulator, operand       # If true, adds the current value of the operand to the result
    SHF  operand, base_1            # Shifts the multiplicand to the left
    SHF  operator, base_n_1         # Shifts the multiplier to the right
    BEQz operator                   # Checks if there are no more operator to multiply
    JP   done                       # Jump to done (used to return from function call)
    JP   loop                       # Repeat the loop
done:
    # Result is in r7
```
> Only operating with shift left (SHL) resulted in the utilization of the last easy to access register (without change register bank), this sacrifice was made to spare the OP Code for a new operation.

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
    j done             # Jump to done (used to return from function call)
done:
    # Result is in x3 (
```

Even though RISC-V has less instruction, it's using 16-bit instruction, so its 7 instructions would translate to 14-Bytes of storage, while on MISA-I with its 8-bit instructions its 10 instructions are only using 10-Bytes... Of course, this pseudo advantage can be denied if taken in consideration that as MISA uses more instructions for preparation (clearing registers and setting values), and also the major performance efficiency advantage that RISC-V does have as its needs to process less instruction making its clock more effective (even a MISA with instruction fusion would have its limitations) is hard to compete. Anyway, a win is a win, let's take anything that is possible! In a multiplication loop, MISA-I is 4-byte shorter!
