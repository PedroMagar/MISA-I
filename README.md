
# MISA-I
**My ISA version 1** is an 8-bit MISC ISA (in spirit) made to be functional, efficient and competitive against well established architectures (AKA z80 / MOS 6502).
>The specification is still in development and there is still a long way to go...

## Architecture
MISA-I is an 8-bit instruction Two-Address architecture, it consist of 1 program counter register, 8 management registers, 16 general purpose registers that are equally divided into 4 registers bank while only two banks are active at a time (making 8 registers addressable) and 1 memory stack address register that operates much like a register bank. Since 8-bit is too little to hold two full addresses for an 8-register architecture without compromising instruction size, the instruction will specify the Rd (register destiny) while Rs (register source) will be relative to it (when F=0 address will be Rs=Rd+4 while F=1 will be Rs=Rd+3).
> With MEM_ADDR been replaced by MEM_STK, this makes possible for the address to utilize as many bits as desired.

### Characteristics:
- One Program Counter (PC) register.
- One Memory Stack (MEM_STK) register:
	- MEM_STK has 4x CPU Operation Size as its size.
	- Operations (STKS/JAL) will only use the lower bits.
	- Stack Rotations (STKR) will shift rotate the register by 'Current Operation Size' bits to the right.
	> MEM_STK brings many advantages over MEM_ADDR, starting by allowing each CPU implementation to decides how many address bits would be appropriate for it, a low bit CPU will have a standard behaviour when its address space is bigger than its registers size, it  would be possible to hold a write, read and/or jump address at the same time, the stack could be used as a extension to the registers file size, also access violation would be easier to debug because the address would be in a know register. The downside of such implementation is that to access its data would be necessary to rotate the register, also split core concept becomes more taxing (a second core would need its own stack).
- 8 Management Registers:
	- r11: Interruption Address.
	- r10: Interruption timer (cycles).
	- r9: Relative Address.
	- r8: Field Length.
	- r7: CPUID - Current CPU capabilities.
	- r6: NPC - New Core Program Counter
	- r5: FMA - Flex Memory Address.
	- r4: INTM - Interruption message.
	> Currently, the focus of 'Management' mode is to access some CPU information such as what is its capabilities or how is its current operation state, there is also an attempt to implement protected memory, this can be seen with r9 and r8, restore instructions (RER) would refuse to restore an "higher privileged" (out of bounds) state, a restore in the max address could also indirectly be used to return control to the caller.
- 16 registers: split in sets of 4 (Register Bank), the instruction field in SRB (Swap Register Bank) will determine which set is to be used, CPU starts with Rb0 and Rb1 active, SRB will match the register bank accordingly:
	- 00 / LL: Will use the default register bank, making avaliable: r7-r4 and r3-r0.
	- 01 / LH: Will swap the lower register bank, making avaliable:  r7-r4 and r11-r8.
	- 10 / HL: Will swap the higher register bank, making avaliable: r15-r12 and r3-r0.
	- 11 / HH: Will swap both registers banks, making avaliable: r15-r12 and r11-r8.
	> A CPU configuration without register bank (8 fixed registers) is under consideration, this could spare some transistor that could be crucial for a truly MOS 6502 competitor.
- Two addresses architecture. **[Rd = Rd (op) Rs]**
- Instructions that operate with two-registers will always assume Rd = RRR from instruction, while Rs is RRR+4 if function bit (F) is 0 or RRR+3 if function bit (F) is 1.
 > Rs address is still under consideration, could be RRR+4 and RRR+1|RRR-1.
- Carry behavior: Instructions that could exceed registers capacity (ADD, SUB, SHF) will fill an internal bit (carry) that can not be read directly but can be used for flow control (branch) with appropriate instruction (BC).
- Branch Behavior: Skip one instruction if false.
  > Behavior is still under review, maybe could skip two instructions (it would allow a SRB or STKS instruction before jump), or maybe could utilize another register for address.
- Operation Mode: If supported, the CPU can split its registers in half (until it reach 8-bits) and do operations on it, the registers composition can be as follows:
	- Management - r11-r4 will hold the CPU configuration. (obrigatory)
	- Full 32 bit mode - the CPU will use the full register size and manipulate data in 32 bit.
	- High 16 bit mode - CPU will only use bits 31 to 16 of its registers and manipulate data in 16 bit.
	- Low 16 bit mode - CPU will only use bits 15 to 0 of its registers and manipulate data in 16 bit.
	- High 16 High 8 bit mode - CPU will only use bits 31 to 24 of its registers and manipulate data in 8 bit.
	- High 16 Low 8 bit mode - CPU will only use bits 23 to 16 of its registers and manipulate data in 8 bit.
	- Low 16 High 8 bit mode - CPU will only use bits 15 to 8 of its registers and manipulate data in 8 bit.
	- Low 16 Low 8 bit mode - CPU will only use bits 7 to 0 of its registers and manipulate data in 8 bit.
	> The smaller size is under consideration to become 4-bits, this would make it friendlier to software BCD.
- CPUID (8-bit mandatory):

| Custom | MIMD  | SIMD-M | SIMD-V | Write Police | Protected Memory | Operations (8/16/32-bit/Unknown) |
|--------|-------|--------|--------|--------------|------------------|----------------------------------|
| 1-bit  | 1-bit | 1-bit  | 1-bit  | 1-bit        | 1-bit            | 2-bit                            |
> **Custom** will replace lower bits for instead of a CPU capabilities, it becomes a true CPU ID that the running application will have to identify.
- Split Core (MIMD):
	- Core will start a new execution loop in parallel on NPC address as soon as NPC is filled with any value besides 0.
- Vector Mode (SIMD-V):
	- Will replace the upper 4 registers with vector registers by a reference of the register bank.
	- RRR F --> VV RR: Where 'RR' is the operator and will be any of the lower register files (r3-r0 or r11-r8) while 'VV' will be the vectorized reference of the full register file:
		- v0: bank 0 - Registers from r0 to r3.
		- v1: bank 1 - Registers from r4 to r7.
		- v2: bank 2 - Registers from r8 to r11.
		- v3: bank 3 - Registers from r12 to r15.
	- RRR ---> RV F: Where 'F' will define if operation is on register file (0) or on the vector registers (1), 'RV' will hold the address of register or vector that the operation must be over.
		- ex: RV F = ```INC 10 1``` will increment v2 (all registers in register bank 2) by 1, while ```INC 10 0``` will increment the current r2 from active register bank.
	- Limitations:
		- STKS, APC and JAL will stay unchanged and operate on the original register name, this way it's possible to keep the control flow working without consuming addresses available to vector operations.
		> Branch (BEQz/BNEz) is under review as a valid operation over the vector register file.
		> SRB is under consideration to use its two bits to select wich Register bank is in the lower reference.
- Multi Interaction Mode (SIMD-M):
	- Will work similarly to vector mode, but the botton side will also be a reference to the register bank, also the operations are now between each register directly.
	- Ex: ```01 00 OP``` will result in
		- ```r4 = r4 (op) r0```
  		- ```r5 = r5 (op) r1```
  		- ```r6 = r6 (op) r2```
		- ```r7 = r7 (op) r3```
	- VV RR --> RBd RBs: Register bank destiny (RBd) can not be equal Register bank source(RBs).
	> Using RBd = RBs is under considerations, this would be the only method to make a register interact with itself.
- Immediate: Value read from instruction memory.
- RRR F: 
	- RRR: holds the operand register address, all changes will be applied to it.
	- F = 0: This will make the operator be operand + 4 (RRR + 4)
	- F = 1: This will make the operator be operand + 3 (RRR + 3)
	- If the instruction does not have a function bit 'F', it will operate solely on operand or assume F=0.
- Sleep will stop at interruption signal, this can be achieved by an interruption pin or interruption timer.
- *Flex Memory*: A small amount (planned 64/128/256 Bytes) of memory that can be used as a cache or as independent memory, it will be located at the end of the addressable space and its size is what is left.
  >*Flex Memory* can not be bigger than bigger than what a register can address, ex.: an 8-bit CPU implementation can have a 16-bit external address bus, but the max FM it can have is what 8-bit can address, since its located at the end of addressable space, the initial part of the address (most significant bit) would be filled with 1.
- *FMA*: Allows the positioning of the flex memory address, any memory that exceeds the addressable limit can be used as a cache.
- *Split Core* is a wild concept where the CPU will start operating 2 PC and executing two flows, the idea is if the CPU has Flex Memory, it could execute a sub-program entirely in it's Flex Memory while keeping working on the main program in the primary Thread.
- *RST*: Will be used as an escape command to run the CPU in a new architecture (something like "CPU will reset if tryed to run unsupported mode"), but also can be used as a "memory flag" to know if the content in the region can be used RER.
  > Flex Memory/FMA, Split Core and RST are experimental concept under review.

## Instructions
The following table lists the architecture current instructions.

|Binary    |Instruction |Description                             |
|----------|------------|----------------------------------------|
|000 0 0000|NOP         |No Operation                            |
|RRR F 0001|AND         |Logic AND                               |
|RRR F 1001|OR          |Logic OR                                |
|RRR F 0101|XOR         |Logic XOR                               |
|RRR F 1101|SHF         |Shift                                   |
|RRR F 0011|ADD         |Addition                                |
|RRR F 1011|SUB         |Subtraction                             |
|RRR F 0111|MUL*        |Multiplication                          |
|RRR F 1111|DIV*        |Division                                |
|RRR 0 0010|CLR         |Clear                                   |
|RRR 1 0010|INV         |Inverse                                 |
|RRR 0 1010|INC         |Increment                               |
|RRR 1 1010|DEC         |Decrement                               |
|RRR 0 0110|LI          |Load Immediate from instruction         |
|RRR 1 0110|LW          |Load Word                               |
|RRR 0 1110|SW          |Store Word                              |
|RRR 1 1110|**MEMCPY*** |**Memory Copy**                         |
|RRR 0 0100|BEQz        |Branch if Equal to Zero                 |
|RRR 1 0100|BNEz        |Branch if Not Equal to Zero             |
|RRR 0 1100|APC         |Add to Program Counter                  |
|RRR 1 1100|SRSTK       |Swap Register with Stack lower bits     |
|FF0 0 1000|SRB         |Swap Register Bank                      |
|F01 0 1000|SRBSTK      |Swap Register Bank with Stack           |
|D11 0 1000|RSTK        |Rotate Stack                            |
|DB0 1 1000|**RRB**     |**Rotate Register Bank**                |
|001 1 1000|JAL         |Jump and Link                           |
|011 1 1000|**BC**      |**Branch if Carry**                     |
|101 1 1000|SVIM*       |Set Vector Interaction Mode             |
|111 1 1000|SMIM*       |Set Multi Interaction Mode              |
|000 1 0000|SLP         |Sleep (Kill core)                       |
|010 1 0000|FENCE*      |Invalidates I.cache and retires D.cache |
|100 1 0000|SDI         |Send Interruption                       |
|110 1 0000|RER         |Restore all Registers from memory       |
|001 1 0000|OPMS*       |Operation Mode Split Precision          |
|011 1 0000|OPMF*       |Operation Mode Fuse Precision           |
|101 1 0000|OPMH*       |Operation Mode Swap Half                |
|111 1 0000|MNG         |Management Mode                         |
|001 0 0000|CSM         |Control Sequential Memory               |
|011 0 0000|CI          |Control Interruption                    |
|101 0 0000|CRS         |Control Rotation and Signal             |
|111 0 0000|CFP*        |Control Float Point operation           |
|010 0 0000|CWD*        |Control Watchdog                        |
|110 0 0000|WDC*        |Clear Watchdog Time                     |
|100 0 0000|RST         |Reset                                   |

Instructions under review:
- \* : Not mandatory instructions.
- **Bold**: Newly added / under review.
- **RST**: Why a reset instruction? Its main focus would be to change the CPU operation mode to a new version (16-bit), if you try to change and the CPU does not support, it would simply reset, also could be used as a memory flag for XJ to know that the memory region to be restored is a valid one.
- **RSTK/RRB**: *'D'* refeers to direction: *'0'* to left/up and *'1'* to right/down. While *'B'* refeers to bank: *'0'* to lower active Rb(r0-r3/r8-r11) and *'1'* to higher active Rb (r4-r7/r12-r15). More information can be found on *'Development'* section.
- FENCE is something good to have, if someone tried a 1000-core cpu, having a way to sync cache can be useful, also a future 16/32-bit evolution that could enter *"8-bit compact mode"* will have a way to stays true to its cache.
- *CFP* is an emergency mesure to have some possibility of float point operations, this certanly is not required for an 8/16-bit CPU, but this instruction is also aiming at a future evolution running on *compact mode* that would need float point.
- BEQ/BGE: MISA initially was designed with BEQ/BGE in mind, currently they were replaced with MUL/DIV instructions, the idea behind this decision is, even though an 8-bit cpu (or first gen 16-bit) does not need hardware multiplication, a more powerful CPU (an i486 competitor) would not be able to be competitive because of a design flaw (no OP code left), in order to keep the architecture 'future' proof, two highly expensive and valuable OP Codes had to be sacrificed, as a bonus now it's possible to use MISA-I as a compact mode for a future MISA-II without a lot of drawback.
- SPC: 'Split Core' concept can currently be achieved by populating NPC with a value, this feature is likely to be removed, doing so would release a management register to hold 'CPU Mode'.
- INV: Invert all bits that are 0 to 1 and 1 to 0, currently there is no instruction to change the endianness of a register.
- ~~BC: 'Branch if Carry' becomes the strongest branch, since instead of the regular jump 2, it can jump to the address on the register, alternatively, this OP Code could be used for a instruction to change the endianness while BC replace SSIM and keeps the usual behaviour of skip one.~~
- **MEMCPY**: Filling space left by BC, **actively looking for a better instruction to replace it**.
- ~~**STKE**: Due to its redundancy, Set Stack Element Active is under consideration to be replaced by Rotate Up/Down Upper/Lower Register Bank (RxxRB).~~

### Development:
Currently there is a feeling of **missing** some **functionalities**, like:
- Become a true MISC architecture: usability takes priority over philosophy, currently the instruction structure are much more RISC like, and some instructions can even been said to be CISC, here are the main culprits:
	- LI: will read the next instruction (8-bit data mode) or even next two instructions (16-bit data mode) as immediate value to store on register, it's not complex, but can be argued to have a variable length instruction size, even so this instruction is here to stay, because 8-bit does not let left much space for an immediate, without it generating an address to read or store a value would became painfully taxing on the runtime.
	- Changing CPU behavior is something to be debated, as if already existing instructions will have a different behavior some may count then as new instructions, or as a complex instruction, or both, this would lead to a violation of philosophy, none less this one is also here to stay, the opcode saved and flexibility added is what gives hope of this ISA be competitive.
	- RER: Problably a CISC instructions, It's great for preemptive multitasking, based on XJ from CDC 7600, this function would work perfectly to store the current task when an interrupt is detected or when the OS is doing a context switch, if this instruction did not exist, as the architecture doesn't have a JI (Jump Immediate) instruction, it would be impossible to restore all the register when returning to the process because the last register to be restored would have to contain the PC address of when the execution was stopped, this would imply to define at high level a "throw away" register that would only be usable when interruption is set off.
	- Send Interruption: Interruption can be interpreted as a one bit signal, it would likely violate the RISC/MISC philosophy of only dealing with memory, but its benefits of working as a mean for multi-core synchronization, bank switch and others tasks cannot be underestimated (even if the higher bits of the address space can be used similarly), all this flexibility for the small price of an almost throw away instruction space can not be classified as anything other than a bargain, this is why it's here and probably to stay.
	- Protected Memory: MISC abolishes MMU, even so I believe it's important to have a way to protect itself from malicious users. I'm not looking at a full MMU, I believe a MPU would be enough, for now I'm looking at RA (Relative Address) and FL (Field Length) of CDC Architecture.
	- **RSTK/RRB**, the pure cheat: Yeap, it's cheating, I needed a way to keep mandatory instructions count below 32 to fill at least one requirement of a MISC architecture, there was no other way, as stated before, usability would take full priority over philosophy, with the current architectural limitation of addressable registers, the possibility of a register bank rotation makes it much easier for registers interactions, solving one of the most pressuing issues of the architecture. But even so, **RRB** is not set in stone, the same result could be achived by swaping register bank with stack, then rotating the stack and swaping it back, **RRB** only exist because of a power consumption concern, but if it end up been not frequently called while requiring a complex circuitry for implementation, it will be removed without a second tough. Currently analizing if a byte swap (BSWAP) like behaviour or Set Stack Element Active (STKE) would gives a better result.
	- **SOLUTIONS**: RER/RERO, Protected Memory, Fence and others non essentials behavior probably will be defined as non-obligatory or even postergated to MISA-II, in this way, MISA-I can be kept clean and MISC conformant if desirable.
- ~~MEMCPY: With the 'Flex Memory' concept, it would help to have an instruction to copy directly the memory without recurring to LW/SW loop.~~ (currently implemented)
- BIT Operations: Currently there is no bit operations, like reading a bit from a register (for a branch) and writing a single bit on a register (flag).
- Flags: A dedicated register for flags and used by bit operations.
- ~~JAL: Jump and Link could be not destructive, in this case it would be possible to swap with SMEM and change SMEM behavior to be based on register file location.~~ (unified jump behaviour)
- Cache and TSO (Total Store Order): Let's start simple, 128 Bytes of storage on a 6T Flip-Flop would result in 6144 Transistors, a MOS 6502 has 3500, Intel 8080 has 6000 and Z80 has 8500 ... For a small 8-bit CPU implementation, cache is useless or at least on the best of the best a really really really HUGE burden, BUT, if low transistor count (small size) is not one of the requirements, and a wide input design (like 4-way decode) is what you are doing, a small cache can be desired, and when there is a cache, there is a possibility of memory mismatch with other devices on the bus, to prevent this, sync could be done by setting the CPU in write through mode and using FENCE instruction in sensible areas.

There are also some operations that have some **high overhead** due to the architecture:
- Branch between values may require a SUB or ADD operation before comparison, due to the destructive nature of the architecture, if the original value of comparison cannot be lost there will be an overhead to copy the main value to an available register before the Branch, if there is no available register at the moment... Well... glhf...
- ~~CP/MV: Currently the operation to Copy was removed to resolve a major oversight of not having the 'Inverse' operation, as a bonus the architecture also received a branch if carry operation, the downside is since the architecture does not have a way to directly access all register, when the data needs to interact with two register that doesn't have a link betwen then, the data has to be moved/copied, now this can lead to a major overhead as currently the only way to copy is to first clear a register then add the value to be moved/copied to it, so the data in the other register may need to be moved or even worse, stored. This may look like a design flaw but it's an architectural decision (won't say a good one), is not all data that needs to communicate with every register and current design allow enough flexibility to be reasonably efficient.~~ (resolved with RRB/RSTK)

### Honorable Mentions
There are some instructions that was once in the isa, but currently were removed due to conflict/priority compared to the current design, but there are also others that are just waiting to be in:

|Binary    |Instruction |Description                             |Candidate|
|----------|------------|----------------------------------------|---------|
|RRR 1 1110|**BSWAP***  |**Byte Swap**                           |         |
|RRR I IIII|**RNG**     |**Generate Random Value**               |Yes      |
|011 0 1000|RRSTK       |Rotate Stack to the Right               |;D       |
|111 0 1000|RLSTK       |Rotate Stack to the Left                |;D       |
|FF0 1 1000|**STKE**    |**Set Stack Element Active**            |YES      |
|000 1 1000|RDLRB       |Rotate Down Lower Register Bank         |;D       |
|100 1 1000|RULRB       |Rotate Up Lower Register Bank           |;D       |
|010 1 1000|RDURB       |Rotate Down Upper Register Bank         |;D       |
|110 1 1000|RUURB       |Rotate Up Upper Register Bank           |;D       |
|011 1 1000|**SSIM***   |**Set Single Interaction Mode**         |-,-      |
|III I IIII|SPC         |Split Core                              |Yes      |
|III I IIII|CSHD        |Control Shift Direction                 |         |
|FFF 1 1000|OPM         |Operation Mode                          |No       |
|RRR 0 1000|SMEM        |Swap Memory Address                     |         |
|RRR 1 1100|JALR        |Jump and Link from register             |         |
|RRR F 0011|SHL         |Shift Left                              |         |
|RRR F 1011|SHR         |Shift Right                             |         |
|RRR 0 0110|ADDC        |ADD Carry                               |         |
|RRR F 0111|BEQ         |Branch if Equal                         |Yes      |
|RRR F 1111|BGE         |Branch if Greater or Equal              |Yes      |
|RRR F 1111|BLT         |Branch if Less Than                     |         |
|RRR F IIII|CBN         |Control Branch Negate                   |         |
|RRR 0 0001|CP          |Copy                                    |         |
|RRR 0 0001|MV          |Move                                    |         |
|110 0 0000|RERO*       |Restore Operation Mode Registers        |         |
|110 0 0000|BKPR        |Restore Backup from memory              |         |
|110 0 0000|FENCE.I     |Invalidates instruction cache           |         |
|110 0 0000|FENCE.D     |Retires data cache                      |         |
|RRR R IIII|BITW        |Write Bit                               |         |
|III I IIII|BITT        |Test Bit                                |         |
|III I IIII|RPC         |Read PC                                 |         |
|III I IIII|RI          |Read Interruption                       |         |
|110 1 0000|IIL         |Invert Interuption Line                 |         |
|III I IIII|CR          |Control Rotation                        |No       |
|III I IIII|CS          |Control Signal                          |No       |
|III I IIII|CC          |Control Carry                           |No       |
|III I IIII|CIO         |Control Interruption on Overflow        |         |
|III I IIII|CPUID       |Read CPU capabilities                   |         |

> **Bold**: Newly added.

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
> Only operating with shift left (SHF) resulted in the utilization of the last easy to access register (without changing register bank), this sacrifice was made to spare the OP Code for a new operation.

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

Even though RISC-V has less instruction, it's at best using 16-bit instruction, so its 7 instructions would translate to 14-Bytes of storage, while on MISA-I with its 8-bit instructions is using 10-Bytes for its 10 instructions... Of course, this pseudo advantage can be denied if taken in consideration that as MISA uses more instructions for preparation (clearing registers and setting values), and also the major performance efficiency advantage that RISC-V does have as its needs to process less instruction making its clock more effective is hard to compete (even a MISA with instruction fusion would have its limitations). Anyway, a win is a win, let's take anything that is possible! In a multiplication loop, MISA-I is 4-byte shorter!
