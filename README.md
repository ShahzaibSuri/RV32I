RV32I Single-Cycle Processor (Logisim)
A complete, single-cycle implementation of the RISC-V RV32I base integer instruction set, built in Logisim as part of a RISC-V processor design project at MERL (Microelectronics Research Lab).
This processor implements the full RV32I compute and control-flow instruction set — arithmetic, logic, shifts, comparisons, loads/stores, branches, jumps, and the U-type instructions — in a classic single-cycle datapath: one instruction fetched, decoded, executed, and written back every clock cycle.
Repository structure
RV32I.circ                          ← top-level datapath (open this file)  
Components/  
  ├── ALU.circ                      ← Arithmetic\_Logic\_Unit  
  ├── Register\_File.circ            ← Register\_File\_32 (32 × 32-bit registers)  
  ├── Control\_Unit.circ             ← Control\_Unit  
  └── Immediate\_Generation.circ     ← Immediate\_Generator
Important: RV32I.circ references the four component files by relative path (Components/ALU.circ, etc.). This folder structure must be preserved exactly when cloning or downloading the repo, or Logisim will fail to resolve the sub-circuits. If you rename any file in Components/, you must also update the corresponding \<lib\> path inside RV32I.circ, or the project will not open.
Requirements
    • Logisim-Evolution (recommended) or Logisim, version compatible with the file format used here.
Getting started
    1. Clone the repository, keeping RV32I.circ and Components/ in the same relative layout shown above.
    2. Open RV32I.circ in Logisim.
    3. Double-click the Instruction_Memory RAM and load your program (hex machine code, one 32-bit word per address) using Logisim's "Load Image" option, or edit it directly in the RAM's hex editor.
    4. Set RST high once to initialize the PC to 0, then release it.
    5. Clock the CLK pin (manually, or via a clock component) to step through the program one instruction per cycle.
    6. Inspect register contents via the Register_File_32 sub-circuit, and memory contents via the Data_Memory RAM.
Architecture overview
Single-cycle datapath — every instruction completes in exactly one clock cycle. Major blocks:
Block	Role
PC (Program Counter)	32-bit register holding the current instruction address; updated every cycle from the Next-PC mux
Instruction Memory	1K × 32-bit RAM (nonvolatile), addressed by PC, holds the program
Control Unit	Decodes the opcode/funct3/funct7 fields and produces every control signal below
Register File	32 × 32-bit general-purpose registers, 2 read ports + 1 write port, x0 hardwired to zero
Immediate Generator	Reconstructs and sign-extends I/S/B/U/J-type immediates from the raw instruction
ALU	Performs all arithmetic, logic, shift, and comparison operations; also produces the branch condition (Branch output)
Data Memory	1K × 32-bit RAM (nonvolatile), used by loads and stores
Adders	Separate dedicated adders for PC+4, and for computing branch/jump/JALR target addresses in parallel with the ALU

Instruction set support
All 37 base RV32I instructions across every format are supported, except FENCE, ECALL, and EBREAK, which are not implemented (no exception/trap handling in this design).
R-Type (opcode 0110011)
Instr	funct3	funct7	ALU Code
ADD	000	0000000	0
SUB	000	0100000	8
SLL	001	0000000	1
SLT	010	0000000	2
SLTU	011	0000000	3
XOR	100	0000000	4
SRL	101	0000000	5
SRA	101	0100000	13
OR	110	0000000	6
AND	111	0000000	7

I-Type — Arithmetic (opcode 0010011)
Instr	funct3	ALU Code
ADDI	000	0
SLTI	010	2
SLTIU	011	3
XORI	100	4
ORI	110	6
ANDI	111	7
SLLI	001	1
SRLI	101 (funct7=0000000)	5
SRAI	101 (funct7=0100000)	13

I-Type — Loads (opcode 0000011)
Instr	funct3	ALU Code
LB	000	0 (address = rs1 + imm)
LH	001	0
LW	010	0
LBU	100	0
LHU	101	0

I-Type — JALR (opcode 1100111)
Instr	funct3	ALU Code
JALR	000	31 (link value via shared adder; target via dedicated JALR-target adder)

S-Type — Stores (opcode 0100011)
Instr	funct3
SB	000
SH	001
SW	010

B-Type — Branches (opcode 1100011)
Instr	funct3	ALU Code
BEQ	000	16
BNE	001	17
BLT	100	20
BGE	101	21
BLTU	110	22
BGEU	111	23

U-Type (opcode alone disambiguates)
Instr	opcode	ALU Code
LUI	0110111	9
AUIPC	0010111	10

J-Type (opcode 1101111)
Instr	ALU Code
JAL	31 (shared adder with JALR)

ALU design
The ALU uses a 5-bit control code (32-way select), rather than the minimal 4-bit/10-op scheme some textbooks use — branches, JAL/JALR, LUI, and AUIPC all have their own dedicated codes inside the ALU itself instead of being handled purely by external logic.
Code (dec)	Instructions	Code (dec)	Instructions
0	ADD, ADDI	13	SRA, SRAI
1	SLL, SLLI	16	BEQ
2	SLT, SLTI	17	BNE
3	SLTU, SLTIU	20	BLT
4	XOR, XORI	21	BGE
5	SRL, SRLI	22	BLTU
6	OR, ORI	23	BGEU
7	AND, ANDI	31	JAL, JALR
8	SUB	9	LUI
—	—	10	AUIPC

Branch detection trick: all six branch codes (16–23) share the same top-2-bit pattern (10), so the ALU's Branch output is derived from a single 2-bit check on the ALU code's upper bits, ANDed with the actual comparison/subtraction result — rather than needing six separate equality checks.
Operand routing (decided outside the ALU, by the Control Unit):
    • Operand A: rs1 (default) / PC (AUIPC) / PC+4 (JAL, JALR link value)
    • Operand B: rs2 (R-type) / immediate
Unused ALU codes (9 and 10 aside): 11, 12, 14, 15, 18, 19, 24–30 are tied to a safe default output.
Control Unit signal table
Type	RegWrite	MemWrite	MemRead	Branch	OpA	OpB	NextPC	ALU Op	ALU Code	ImmSel
R-Type	1	0	0	0	rs1	rs2	PC+4	0	funct3/7	—
I-Type	1	0	0	0	rs1	imm	PC+4	1	funct3	I
Loads	1	0	1	0	rs1	imm	PC+4	2	0 (ADD)	I
Stores	0	1	0	0	rs1	imm	PC+4	4	0 (ADD)	S
Branches	0	0	0	1	rs1	rs2	Branch-target adder	5	>16	—
LUI	1	0	0	0	—	imm	PC+4	6	9	U
AUIPC	1	0	0	0	PC	imm	PC+4	7	10	U
JALR	1	0	0	0	PC+4	0 (forced)	JALR-target adder	3	31	0
JAL	1	0	0	0	PC+4	0 (forced)	Jump-target adder	3	31	0

Next PC selection (outside this table, at the PC mux): PC+4 by default; branch-target adder's output if Branch is asserted by both the CU and the ALU's condition result; the dedicated jump-target adder for JAL; the dedicated JALR-target adder ((rs1+imm) & ~1) for JALR.
Memory
    • Instruction Memory: 1K × 32-bit words (4 KB), nonvolatile RAM, addressed by PC
    • Data Memory: 1K × 32-bit words (4 KB), nonvolatile RAM, addressed by ALU-computed address
Known limitations
    • Single-cycle only — no pipelining, no hazard handling needed as a result, but also no performance optimization
    • FENCE, ECALL, EBREAK are not implemented — no memory-ordering, syscall, or breakpoint/trap support
    • No exception/interrupt handling
    • Byte/halfword load-store width handling (LB/LH/LBU/LHU/SB/SH) depends on the Data Memory's access-width configuration — verify this matches your RAM component's settings before relying on sub-word memory operations
