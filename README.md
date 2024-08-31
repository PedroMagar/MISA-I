# MISA-I
**My ISA version 1** is a 8-bit MISC (stretching its meaning) compatible ISA (in the end, the implementation is what matters) made to be functional, efficient and competitive against well established architectures (AKA z80 / MOS 6502). The goal is to see how much could be improved upon the aged ISA's if they were designed with a modern concept in mind.
>The specification is still in development and there is still a long way to go...

## Architecture
The architecture will consist of 12 registers while 8 are addressable at a time. Since 8-bit instruction is too little to hold two full addresses for an 8-register architecture without compromising instruction size, the instruction will specify the Rd (register destiny) while Rs (register source) will be relative to it.

## Instructions
The following table lists the architecture current instructions.

|Binary    |Instruction |Description                       |
|----------|------------|----------------------------------|
|000 0 0000|NOP         |                                  |
|RRR F 0001|CP          |Copy                              |
|RRR F 1001|AND         |                                  |
|RRR F 0101|OR          |                                  |
|RRR F 1101|XOR         |                                  |
|RRR F 0011|SL          |Shift Left                        |
|RRR F 1011|SR          |Shift Right                       |
|RRR F 0111|BEQz        |Branch Equal Zero                 |
|RRR F 1111|BGEz        |Branch Greater Equal Zero         |
|RRR F 0010|ADD         |                                  |
|RRR F 1010|SUB         |                                  |
|RRR 1 0110|INC         |Increment                         |
|RRR 0 0110|CLR         |Clear                             |
|RRR 1 1110|DEC         |Decrement                         |
|RRR 0 1110|LI          |Load Immediate from instruction   |
|RRR 0 0100|LW          |Load Word                         |
|RRR 1 0100|SW          |Store Word                        |
|RRR F 1100|SMEM        |Swap Memory Address               |
|RRR 0 1000|APC         |Add to Program Counter            |
|RRR 1 1000|JAL         |Jump and Link                     |
|XX0 1 0000|**TO-DO**   |Open Space for new instruction    |
|001 1 0000|CSM         |Control Sequential Memory         |
|011 1 0000|CI          |Control Interruption              |
|101 1 0000|CA          |Control Array                     |
|111 1 0000|SLP         |Sleep                             |
|II1 0 0000|OPM         |Operation Mode                    |
|010 0 0000|BKP         |Backup Registers to memory        |
|110 0 0000|BKPR        |Restore Backup from memory        |
|100 0 0000|CWDT        |Clear Watchdog Time               |

### Development: Current missing
Currently there is a feeling of missing some functionalities, like:
- Reading (branch) and writing a single bit of a register (flag).
- Vector operations (even without multiply).
- Flags: Can be used to read the processor current operation mode without going to management mode.
- JAL: Jump and Link could be not destructive, in this case would be possible to swap with SMEM and change SMEM behavior to be based on register file location.

There are also some operations that have some high overhead due to the architecture:

- Branch between values may require a SUB or ADD operation before comparison, due to the destructive nature of the architecture, if the original value of comparison cannot be lost there will be an operation overhead to copy the main value to an available register before the Branch, if there is no available register at the moment... Well... glhf...

### Design: Characteristics
There are some design behavior mandate by the isa that every implementation must follow:
- Immediate: Currently only used on LWI, Immediate is obtained by reading the following bits from instruction as immediate instead of executing it as instruction.



### Honorable Mentions
There are some instructions that was once in the isa, but currently were removed due to conflict/priority compared to the current design, but there are also others that are just waiting to be in:

|Binary    |Instruction |Description                       |Candidate|
|----------|------------|----------------------------------|---------|
|RRR 0 1110|FILL        |Fill register with 1              |         |
|RRR 0 1110|SWPF        |Swap Flags                        |         |
|III I IIII|RST         |Reset                             |Yes      |
|III I IIII|SLPI        |Sleep until Interrupt             |         |
|RRR I IIII|BC          |Branch if carry                   |         |
|RRR R IIII|BITW        |Write Bit                         |         |
|III I IIII|BITT        |Test Bit                          |         |
|III I IIII|MKA         |Make Array                        |Yes      |
|III I IIII|RPC         |Read PC                           |         |
|III I IIII|RI          |Read Interruption                 |         |
|RRR I IIII|RNG         |Generate Random Value             |         |
|III I IIII|CR          |Control Rotation                  |Yes      |
|III I IIII|CS          |Control Signaled                  |Yes      |
|III I IIII|CIO         |Control Interruption on Overflow  |Yes      |
|III I IIII|CC          |Control Carry                     |Yes      |
|III I IIII|MP          |Make Process                      |         |
|III I IIII|RPR         |Reserve Private Memory            |         |
|III I IIII|RPU         |Reserve Public Memory             |         |
|III I IIII|EADD        |Exception Address                 |         |
|III I IIII|CPUID       |Read CPU capabilities             |Yes      |
