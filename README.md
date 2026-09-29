# Updated-Understanding-the-SERV-RISC-V-Core-
  In this repository we will be understanding the architecture and internal working of the SERV (Serial RISC-V) core.
  
  The goal is not just to use the core, but to go through its RTL implementation and understand how a RISC-V processor can be built from the ground up, especially with SERV's unique serial architecture.

## About SERV

  SERV stands for SERial RISC-V. It is a small, resource-efficient RISC-V processor core designed by Olof Kindgren.
  
  Unlike a conventional RISC-V CPU that processes a complete word in parallel, SERV processes data one bit at a time. This makes the design extremely small and simple, at the cost of performance.
  
  SERV is therefore a great project for learning:
  
    1. RISC-V processor architecture
    2. CPU datapaths and control logic
    3. Instruction execution
    4. Register files
    5. ALUs
    6. Program counters
    7. Memory interfaces
    8. RTL designVerilog/SystemVerilog
    9. FPGA implementation
    10. Bit-serial processor architectures

What Are We Understanding?

Going through the SERV source code and trying to understand the complete path of an instruction through the processor.

Some of the questions we are exploring—along with detailed technical explanations from the RTL—include:

1. How is a RISC-V instruction fetched?
   
   RTL Implementation: SERV maintains the Program Counter (PC) value and sends it out via an instruction memory interface (often wrapped in a simple Wishbone bus protocol).
   The Process: The memory or instruction cache returns a 32-bit instruction word. Because SERV is bit-serial in its execution, the fetched 32-bit instruction is typically captured and stored locally in instruction decode registers so that fields can be sliced or scanned as needed by the control logic over multiple execution cycles.

2. How is the instruction decoded?

      RTL Implementation: The instruction word is parsed using standard RISC-V bitfield extractions:

      opcode: bits [6:0]
      rd: bits [11:7]
      funct3: bits
      rs1: bits
      rs2: bits
      funct7: bits

      Control Mapping: These fields feed directly into the central state machine (serv_controller or equivalent FSM blocks) to assert control lines for ALU operations, register file addressing, and memory read/write requests.

3. How are registers read and written?

   RTL Implementation: Unlike traditional register files built from large arrays of parallel flip-flops or RAM blocks with independent multi-port decoders, the SERV register file (serv_rf_top and related modules) is designed around shift registers.

   Serial Access:
         
       To read, the register index (rs1 or rs2) is selected, and the 32-bit value streams out bit-by-bit over 32 clock cycles (starting from bit 0 up to bit 31).

       To write, incoming result bits are shifted back into the addressed destination register (rd) one bit per clock cycle.

   The Zero Register ($x0$): Hardwired to zero. Logic blocks intercept reads to $x0$ to immediately stream out zeros, and writes to $x0$ are safely ignored/discarded.

4. How does the ALU perform operations one bit at a time?

   RTL Implementation: The serial ALU (serv_alu) contains primitive combinatorial logic (AND, OR, XOR, full adders) operating on a single bit slice per cycle.

   Carry Management: For arithmetic operations like addition and subtraction, a carry-out bit is computed alongside the sum bit for the current bit position. This carry-out is registered and fed back as the carry-in for the next clock cycle (representing the next bit position, from bit 0 to bit 31).
   
5. How does the program counter work?

   RTL Implementation: The Program Counter (serv_pc) tracks the execution address. For sequential execution, it increments by 4 bytes (or handles compressed instructions if enabled, though base SERV focuses on RV32E/I).

   Calculations: During branches or jumps, target offsets (like B-type or J-type immediates) are streamed into the ALU sequentially to be added to the current PC or base register, updating the PC target over multiple cycles.

6. How are branches and jumps handled?

   RTL Implementation: Conditional branches (beq, bne, blt, etc.) compare two register values bit-by-bit through the serial ALU. Because comparison can sometimes terminate early or require scanning through bit 31, the control logic waits for the appropriate cycle count, evaluates the condition flag, and updates the PC if the branch is taken. Jumps (jal, jalr) calculate the target address in a similar fashion while saving the return address (PC + 4) into the link register (rd).

7. How does the core communicate with memory?

   RTL Implementation: SERV interfaces with memory using a minimal control/data handshaking protocol (such as a serial or reduced Wishbone bus). Control signals dictate read/write enablement, while address and data lines accommodate the serial nature of the processor where necessary.

8. How are loads and stores implemented?

   RTL Implementation:
     Stores: Data from rs2 is streamed out bit-by-bit to the memory data bus. Byte-enable signals are generated based on the lower bits of the computed memory address and the instruction type (sb, sh, sw).
     Loads: Data returned from memory is captured. Depending on whether it is a byte, half-word, or full word (and whether it is signed or unsigned via lbu, lh, etc.), the core handles alignment, sign-extension, and routing back to the register file.

9. How does the control logic coordinate all these operations?

   RTL Implementation: A centralized Finite State Machine (FSM) acts as the heart of SERV. It steps through phases: instruction fetch, operand read setup, execution loop (counting 0 to 31 for the 32 bit-serial cycles), memory access (if applicable), and write-back.

10. Why was a serial architecture chosen?

    Design Philosophy: To achieve the absolute minimum hardware footprint possible. By reusing a 1-bit datapath for 32 iterations instead of replicating parallel 32-bit hardware paths, transistor count and FPGA resource consumption drop dramatically.

11. What are the trade-offs between area, speed, and complexity?

    Area vs. Performance: SERV trades raw execution speed (taking multiple cycles per instruction due to bit-serial serialization) for an ultra-small area footprint, making it a masterclass in constrained digital design.

Architecture :

  At a high level, we study SERV as a collection of interacting blocks:
  
                  +----------------------+
                  |      RISC-V          |
                  |    Instruction       |
                  +----------+-----------+
                             |
                             v
                  +----------------------+
                  |   Instruction        |
                  |     Decode           |
                  +----------+-----------+
                             |
                             v
          +------------------+------------------+
          |                                     |
          v                                     v
   +---------------+                     +---------------+
   | Register File |                     |  Control /    |
   |               |                     |   Sequencing  |
   +-------+-------+                     +-------+-------+
           |                                     |
           +------------------+------------------+
                              |
                              v
                   +----------------------+
                   |    Serial Datapath   |
                   |                      |
                   |  ALU / Shifters /    |
                   | Address Generation   |
                   +----------+-----------+
                              |
                              v
                   +----------------------+
                   |   Memory Interface   |
                   +----------------------+

  The actual SERV implementation is more specialized than this simplified diagram, and one of the main goals of this repository is to understand how the different RTL modules fit together.

Why a Serial CPU?

  A normal CPU datapath might operate on 32 bits in parallel. SERV takes a different approach. Instead of performing an operation on all 32 bits simultaneously, the datapath processes the bits serially.
  
  For example, conceptually:
     
     32-bit operation
     
     Parallel CPU:
       +---+---+---+---+---+---+---+---+
       |31 |30 |29 |...| 3 | 2 | 1 | 0 |
       +---+---+---+---+---+---+---+---+
                       |
                       v
              Process in parallel
     
     
     SERV:
     
       bit 0 -> bit 1 -> bit 2 -> ... -> bit 31
                  |
                  v
              Process serially
              
This significantly reduces the amount of hardware required. The trade-off is that an operation takes multiple clock cycles. Understanding this trade-off is one of the key takeaways from studying SERV.

Learning Approach :

  The general approach is:
    1. Understand the RISC-V ISA concepts required by the core.
    2. Identify the major RTL modules.
    3. Understand the signals between those modules.
    4. Follow the execution of an instruction cycle by cycle.
    5. Understand how the serial datapath processes individual bits.
    6. Trace arithmetic, logical, load/store, and branch instructions.
    7. Understand how the program counter changes.
    8. Understand how memory accesses are generated.
    9. Relate the RTL implementation back to the RISC-V specification.
    10. Document findings as we go.

Things Plan to Document :

1. Instruction Fetch

   Understanding how SERV obtains an instruction from memory and how the instruction reaches the execution logic.

3. Instruction Decode

   Understanding how the fields of a RISC-V instruction are interpreted:Plaintext

     +---------+---------+---------+---------+---------+---------+
     | funct7  |   rs2   |   rs1   | funct3  |   rd    | opcode  |
     +---------+---------+---------+---------+---------+---------+
   
We connect these instruction fields to the corresponding control signals in the RTL.

3. Register File
   Understanding:
     How registers are stored (using shift-register topologies)
     How registers are read serially over 32 cycles
     How registers are written back bit-by-bit
     How the zero register ($x0$) is handled
     How serial processing interacts with the register file
4. Serial ALU

   One of the main areas we want to understand is how common operations are implemented using a bit-serial datapath:

      Operand A
          |
          v
      +-------+
      |       |
      |  ALU  | ---> Result bit
      |       |
      +-------+
          ^
          |
      Operand B
      
Rather than calculating the complete result in one cycle, the datapath works through the bits over multiple cycles with feedback carry registers.

5. Program Counter
    Investigating how the PC is updated for:
      Sequential execution (incrementing)
      Conditional branches
      Jumps (jal)
      Jump-and-link instructions (jalr)
      Exceptions or other control-flow changes where applicable

6. Load and Store Instructions

   Understanding how memory addresses are generated and how data moves between the processor and memory, including how multi-byte values are handled despite the serial nature of the core.7. Control and SequencingA major part of understanding SERV is understanding when each operation happens. We trace internal control signals and state transitions to see how a single RISC-V instruction is broken into multiple steps.Example Instruction TracingPlaintextInstruction:
    ADD x3, x1, x2

Goal:
    x3 = x1 + x2

Questions to trace:

  1. How is the instruction fetched? (PC points to address; instruction is fetched via bus interface).
  2. How is ADD identified? (Opcode, funct3, and funct7 bitfields are decoded by the FSM).
  3. How are x1 and x2 selected? (rs1 and rs2 indices point to respective register shift chains).
  4. How does the register file provide the operands? (Bits stream out sequentially from cycle 0 to 31).
  5. How does the serial ALU perform the addition? (Full adder processes bit `i` with carry-in from bit `i-1`).
  6. How many cycles are required? (32 data cycles plus control overhead states).
  7. How is the result written back to x3? (Result bits stream back into the destination register chain).
  8. What happens to the program counter? (PC updates to the next instruction address).
    
  The idea is to follow an instruction from fetch → decode → execute → memory (if required) → write-back.

## RISC-V ISA :

SERV implements a small RISC-V ISA subset (RV32E/RV32I), making it particularly interesting for understanding the relationship between an ISA and its hardware implementation. As part of this project, we study relevant RISC-V instructions and map them directly to the RTL implementation, focusing on the hardware rather than simply memorizing instructions.Why SERV?SERV demonstrates that a CPU does not necessarily need a large, complicated parallel datapath. A processor can be constructed using a very small amount of hardware by trading parallelism and performance for extreme simplicity and area efficiency, making it an invaluable design to study for mastering the fundamentals of CPU architecture.Repository Status

## 🚧 Work in Progress :

  This repository is primarily a learning and documentation project. Understanding evolves as we dig deeper into the RTL, and explanations are continuously updated or corrected as we verify them against the actual implementation.
