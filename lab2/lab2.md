---
Authors: Jason Lowe-Power, Filipe Eduardo Borges
Editor: Justin Perona
Title: ECS 154B Lab 2, Winter 2020
---

# ECS 154B Lab 2, Winter 2020

**Due by 11:59 PM on January 27, 2020.**

*Turn in via Gradescope*.
[See below for details.](#Submission)

# Table of Contents

* [Introduction](#introduction)
  * [How this assignment is written](#how-this-assignment-is-written)
  * [Goals](#goals)
* [Single cycle CPU design](#single-cycle-cpu-design)
* [Control unit overview](#control-unit-overview)
* [Part I: R-types](#part-i-r-types)
  * [R-type instruction details](#r-type-instruction-details)
  * [Testing the R-types](#testing-the-r-types)
* [Part II: I-types](#part-ii-i-types)
  * [I-type instruction details](#i-type-instruction-details)
  * [Testing the I-types](#testing-the-i-types)
* [Part III: `lw`](#part-iii-lw)
  * [`lw` instruction details](#lw-instruction-details)
  * [Testing `lw`](#testing-lw)
* [Part IV: U-types](#part-iv-u-types)
  * [`lui` instruction details](#lui-instruction-details)
  * [`auipc` instruction details](#auipc-instruction-details)
  * [Testing the U-types](#testing-the-u-types)
* [Part V: `sw`](#part-v-sw)
  * [`sw` instruction details](#sw-instruction-details)
  * [Testing `sw`](#testing-sw)
* [Part VI: Other memory instructions](#part-vi-other-memory-instructions)
  * [Other memory instruction details](#other-memory-instruction-details)
  * [Testing the other memory instructions](#testing-the-other-memory-instructions)
* [Part VII: Branch instructions](#part-vii-branch-instructions)
  * [Branch instruction details](#branch-instruction-details)
  * [Branch control unit](#branch-control-unit)
    * [Testing your branch control unit](#testing-your-branch-control-unit)
* [Part VIII: `jal`](#part-viii-jal)
  * [`jal` instruction details](#jal-instruction-details)
  * [Testing `jal`](#testing-jal)
* [Part IX: `jalr`](#part-ix-jalr)
  * [`jalr` instruction details](#jalr-instruction-details)
  * [Testing `jalr`](#testing-jalr)
* [Part X: Full applications](#part-x-full-applications)
  * [Testing full applications](#testing-full-applications)
* [Feedback](#feedback)
* [Grading](#grading)
* [Submission](#submission)
  * [Code portion](#code-portion)
  * [Feedback form](#feedback-form)
  * [Academic misconduct reminder](#academic-misconduct-reminder)
  * [Checklist](#checklist)
* [Hints](#hints)
  * [`printf` debugging](#printf-debugging)
  * [Common errors](#common-errors)

# Introduction

![Cute Dino](../dino-resources/dino-128.png)

In the last assignment, you implemented the ALU control and incorporated it into the DINO CPU to test some bare-metal R-type RISC-V instructions.
In this assignment, you will implement the branch-control and the main control unit.
After implementing the individual components and successfully passing all individual component tests, you will combine these along with the other CPU components to complete the single-cycle DINO CPU.
The simple in-order CPU design is based closely on the CPU model in Patterson and Hennessey's Computer Organization and Design.

This is the first time we have used this set of assignments, so there may be mistakes.
To offset this, we will be offering a variety of ways to get extra credit.
See [the extra credit page](../extra-credit.md) for an up-to-date list of ways to get extra credit.

## Updating the DINO CPU code

The DINO CPU code must be updated before you can run lab 2. We have made the following changes:
- Add the lab2 tests (if you don't see the file Lab2Tests.scala then you don't have the up to date version)
- The solution for cpu.scala for Lab 1 is included
- The solution for the ALU control for Lab1 PLUS the extra inputs is included
- The control unit includes the control signals for the R-type instructions

If you're not a git expert, I suggest you check out the great [Pro Git book](https://git-scm.com/book/en/v2).
However, the code below should get you started.
**Caution**: I have not tested the code below.
I wrote this off the top of my head.
You can always create a copy of your git repo before beginning!

![ If that doesn't fix it, git.txt contains the phone number of a friend of mine who understands git. Just wait through a few minutes of 'It's really pretty simple, just think of branches as...' and eventually you'll learn the commands that will fix everything.](https://imgs.xkcd.com/comics/git.png)
https://xkcd.com/1597/

If you don't want to use your code from lab 1, but want to use my code to get started run the following git commands in the dinocpu/ directory:

```
git checkout -b my-lab1-solution
git add .
git commit -m "Add my solution to lab 1"
git checkout master
git pull
```

The commands above create a new branch (my-lab1-solution) and then commit all of your current outstanding changes to that branch.
Then, it checks out the master branch and pulls the updates from the dinocpu repository (on jlpteaching).

If you have already committed things to master you can use the following:

```
git checkout -b my-lab1-solution
git checkout master
git reset --hard origin/master
git pull
```

This creates a new branch (my-lab1-solution) then resets your master branch to be the same as the origin's master branch.
Finally, it pulls any updates from the origin (jlpteaching/dinocpu, presumably).

If you want to use your own solution to lab 1 it's a little more complicated:

```
git add .
git commit -m "Add my lab1 solution"
git fetch
git merge origin/master
```

This will merge the updates in origin (jlpteaching/dinocpu) into your master branch.
**There will be merge conflicts** since both you and I modified cpu.scala and alucontrol.scala.
You will have to decide how to deal with those conflicts.
See https://help.github.com/articles/resolving-a-merge-conflict-using-the-command-line/ for more information.
You could also do the same as the first two options to create a my-lab1-solution branch and set master to track origin/master directly then merge master with my-lab1-solution.

There are many other ways to deal with this in git. These are the simplest that I could come up with off the top of my head. I'm happy to help out more in office hours if you get stuck. However, debugging git over piazza is going to be tough :). We'll do our best, though!

## How this assignment is written

The goal of this assignment is to implement a single-cycle RISC-V CPU which can execute all of the RISC-V integer instructions.
Through the rest of this assignment, [Part I](#part-i-r-type) through [Part X](#part-x-full-applications), you will implement all of the RISC-V instructions, step by step.

If you prefer, you can simply skip to the end and implement all of the instructions at once, then run all of the tests for this assignment via the following command.
You will also use this command to test everything once you believe you're done.

```
sbt:dinocpu> test
```

We are making one major constraint on how you are implementing your CPU.
**You cannot modify the I/O for any module**.
We will be testing your control unit with our data path, and our data path with your control unit.
Therefore, you **must keep the exact same I/O**.
You will get errors on Gradescope (and thus no credit) if you modify the I/O.

## Goals

- Learn how to implement a control and data path in a single cycle CPU.
- Learn how different RISC-V instructions interact in the control and data path of a single cycle CPU.

# Single cycle CPU design

Below is a diagram of the single cycle DINO CPU.
This diagram includes all of the necessary data path wires and MUXes.
However, it is missing the control path wires.
This figure has all of the MUXes necessary, but does not show which control lines go to which MUX.
**Hint**: the comments in the code for the control unit give some hints on how to wire the design.

In this assignment, you will be implementing the data path shown in the figure below, implementing the control path for the DINO CPU, and wiring up the control path.
You can extend your work from [Lab 1](../lab1.md), or you can take the updated code from [GitHub](https://github.com/jlpteaching/dinocpu/).
You will be implementing everything in the diagram in Chisel (the `cpu.scala` file only implements the R-type instructions), which includes the code for the MUXes.
Then, you will wire all of the components together.
You will also implement the [control unit](#control-unit-overview) and the [branch control unit](#branch-control-unit).

![Single cycle DINO CPU without control wires](../dino-resources/single-cycle.svg)

# Control unit overview

In this part, you will be implementing the main control unit in the CPU design.
The control unit is used to determine how to set the control lines for the functional units and the the multiplexers.

The control unit takes a single input, which is the 7-bit `opcode`.
From that input, it generates the 9 control signals listed below as output.

```
branch:    true if branch or jump and link register (jal). update PC with immediate
memread:   true if we should read from memory
toreg:     0 for writing ALU result, 1 for writing memory data, 2 for writing pc + 4
add:       true if the ALU should add the results
memwrite:  true if writing to the data memory
regwrite:  true if writing to the register file
immediate: true if using the immediate value
alusrc1:   0 for read data 1, 1 for the constant zero, 2 for the PC
jump:      0 for no jump, 2 for jump, 3 for jal (jump and link register)
```

The following table specifies the `opcode` format and the control signals to be generated for some of the instruction types.

| opcode  |opcode format| branch | memread | toreg |   add  | memwrite | immediate | regwrite | alusrc1 | jump |
|---------|-------------|--------|---------|-------|--------|----------|-----------|----------|---------|------|
|    -    |   default   | false  |  false  |   3   |  false |  false   |   false   |   false  |    0    |   0  |
| 0000000 |   invalid   | false  |  false  |   0   |  false |  false   |   false   |   false  |    0    |   0  |
| 0110011 |      R      | false  |  false  |   0   |  false |  false   |   false   |   true   |    0    |   0  |

We have given you the control signals for the R-type instructions.
You must fill in all of the other instructions in the table in `src/main/scala/components/control.scala`.
Notice how the third line of the table (under the `// R-type`) is an exact copy of the values in this table.

Given the input opcode, you must generate the correct control signals.
The template code from `src/main/scala/components/control.scala` is shown below.
You will fill in where it says *Your code goes here*.

```
// Control logic for the processor

package dinocpu

import chisel3._
import chisel3.util.{BitPat, ListLookup}

/**
 * Main control logic for our simple processor
 *
 * Output: branch, true if branch or jump and link register (jal). update PC with immediate
 * Output: memread, true if we should read from memory
 * Output: toreg, 0 for writing ALU result, 1 for writing memory data, 2 for writing pc + 4
 * Output: add, true if the ALU should add the results
 * Output: memwrite, true if writing to the data memory
 * Output: regwrite, true if writing to the register file
 * Output: immediate, true if using the immediate value
 * Output: alusrc1, 0 for read data 1, 1 for the constant zero, 2 for the PC
 * Output: jump, 0 for no jump, 2 for jump, 3 for jal (jump and link register)
 *
 * For more information, see section 4.4 of Patterson and Hennessy.
 * This follows figure 4.22.
 */

class Control extends Module {
  val io = IO(new Bundle {
    val opcode = Input(UInt(7.W))

    val branch = Output(Bool())
    val memread = Output(Bool())
    val toreg = Output(UInt(2.W))
    val add = Output(Bool())
    val memwrite = Output(Bool())
    val regwrite = Output(Bool())
    val immediate = Output(Bool())
    val alusrc1 = Output(UInt(2.W))
    val jump    = Output(UInt(2.W))
  })

  val signals =
    ListLookup(io.opcode,
      /*default*/           List(false.B, false.B, 3.U,   false.B, false.B,  false.B, false.B,    0.U,    0.U),
      Array(                 /*  branch,  memread, toreg, add,     memwrite, immediate, regwrite, alusrc1,  jump */
      // R-format
      BitPat("b0110011") -> List(false.B, false.B, 0.U,   false.B, false.B,  false.B, true.B,     0.U,    0.U),

      // Your code goes here.
      // Remember to make sure to have commas at the end of each line, except for the last one.

      ) // Array
    ) // ListLookup

  io.branch := signals(0)
  io.memread := signals(1)
  io.toreg := signals(2)
  io.add := signals(3)
  io.memwrite := signals(4)
  io.immediate := signals(5)
  io.regwrite := signals(6)
  io.alusrc1 := signals(7)
  io.jump := signals(8)
}
```

In this code, you can see that the `ListLookup` looks very similar to the table above.
You will be filling in the rest of the lines of this table.
As you work through each of the parts below, you will be adding a line to the table.
You will have one line for each type of instruction (i.e., each unique opcode that for the instructions you are implementing).

**Important: DO NOT MODIFY THE I/O.**
You do not need to modify any other code in this file other than the `signals` table!

# Part I: R-types

In the last assignment, you implemented a subset of the RISC-V data path for just R-type instructions.
This did not require a control unit since there were no need for extra MUXes.
In this assignment, you will be implementing the rest of the RISC-V instructions, so you will need to use the control unit.

The first step is to hook up the control unit and get the R-type instructions working again.
You shouldn't have to change all that much code in `cpu.scala` from the first assignment.
All you have to do is to hook up the `opcode` to the input of the control unit.
We have already implemented the R-type control logic for you.
You can also use the appropriate signals generated from the control unit (e.g., `regwrite`) to drive your data path.

## R-type instruction details

The following table shows how an R-type instruction is laid out:

| 31-25  | 24-20 | 19-15 | 14-12   | 11-7 | 6-0     | Name   |
|--------|-------|-------|---------|------|---------|--------|
| funct7 | rs2   | rs1   | funct3  | rd   | 0110011 | R-type |

Each instruction has the following effect.
`<op>` is specified by the `funct3` and `funct7` fields.
`R[x]` means the value stored in register x.

```
R[rd] = R[rs1] <op> R[rs2]
```

## Testing the R-types

You can run the tests for this part with the following command:

```
sbt:dinocpu> testOnly dinocpu.SingleCycleRTypeTesterLab2
```

# Part II: I-types

Next, you will implement the I-type instructions.
These are mostly the same as the the R-types, except that the second operand comes from the immediate value contained within the instruction, rather than another register.

To implement the I-types, you should first extend the table in `control.scala`.
Then you can add the appropriate MUXes to the CPU (in `cpu.scala`) and wire the control signals to those MUXes.
**HINT**: You only need one extra MUX, compared to your R-type-only design.

## I-type instruction details

The following table shows how an I-type instruction is laid out:

|31-20      | 19-15 | 14-12  | 11-7 | 6-0     | Name   |
|-----------|-------|--------|------|---------|--------|
| imm[11:0] | rs1   | funct3 | rd   | 0010011 | I-type |

Each instruction has the following effect.
`<op>` is specified by the `funct3` field.

```
R[rd] = R[rs1] <op> immediate
```

## Testing the I-types

You can run the tests for this part with the following command:

```
sbt:dinocpu> testOnly dinocpu.SingleCycleITypeTesterLab2
```

# Part III: `lw`

Next, we will implement the `lw` instruction.
Officially, this is a I-type instruction, so you shouldn't have to make too many modifications to your data path.

As with the previous parts, first update your control unit to assert the necessary control signals for the `lw` instruction, then modify your CPU data path to add the necessary MUXes and wire up your control.
For this part, you will have to think about how this instruction uses the ALU.
You will also need to incorporate the data memory into your data path, starting with this instruction.

## `lw` instruction details

The following table shows how the `lw` instruction is laid out:

| 31-20     | 19-15 | 14-12 | 11-7 | 6-0     | Name   |
|-----------|-------|-------|------|---------|--------|
| imm[11:0] | rs1   | 010   | rd   | 0000011 | lw     |

`lw` stands for "load word".
The instruction has the following effect.
`M[x]` means the value of memory at location x.

```
R[rd] = M[R[rs1] + immediate]
```

## Testing `lw`

You can run the tests for this part with the following command:

```
sbt:dinocpu> testOnly dinocpu.SingleCycleLoadTesterLab2
```

# Part IV: U-types

U-types are another type of instruction that look similar to the I-types.
There are two of them you need to implement, described below.

## `lui` instruction details

The following table shows how the `lui` instruction is laid out.


| 31-12      | 11-7 | 6-0     | Name   |
|------------|------|---------|--------|
| imm[31:12] | rd   | 0110111 | lui    |

`lui` stands for "load upper immediate."
The instruction has the following effect.
As in C and C++, the `<<` operator means bit shift left by the number specified.

**Important**: The immediate generator will produce the shifted and sign extended value!
You do not need to shift the immediate value outside of the immediate generator.

```
R[rd] = imm << 12
```

## `auipc` instruction details

The following table shows how the `auipc` instruction is laid out.

| 31-12      | 11-7 | 6-0     | Name   |
|------------|------|---------|--------|
| imm[31:12] | rd   | 0010111 | auipc  |

`auipc` stands for "add upper immediate to pc."
The instruction has the following effect.

```
R[rd] = pc + imm << 12
```

## Testing the U-types

You can run the tests for this part with the following command:

```
sbt:dinocpu> testOnly dinocpu.SingleCycleUTypeTesterLab2
```

# Part V: `sw`

`sw` is similar to `lw` in function, but looks closer to an I-type.
You'll need to think about how to implement the changes needed for the data memory.

## `sw` instruction details

The following table shows how the `sw` instruction is laid out.

| 31-25     | 24-20 | 19-15 | 14-12 | 11-7     | 6-0     | Name   |
|-----------|-------|-------|-------|----------|---------|--------|
| imm[11:5] | rs2   | rs1   | 010   | imm[4:0] | 0100011 | sw     |

`sw` stands for "store word."
The instruction has the following effect.
(Careful, while this looks similar to `lw`, it has a very different effect!)

```
M[R[rs1] + immediate] = R[rs2]
```

## Testing `sw`

You can run the tests for this part with the following command:

```
sbt:dinocpu> testOnly dinocpu.SingleCycleStoreTesterLab2
```

# Part VI: Other memory instructions

We now move on to the other memory instructions.
Make sure your `lw` and `sw` instructions work before moving on to this part.

## Other memory instruction details

The following table show how the other memory instructions are laid out.
`lw` and `sw` are included again as a reference.

| 31-25     | 24-20    | 19-15 | 14-12 | 11-7     | 6-0     | Name   |
|-----------|----------|-------|-------|----------|---------|--------|
| imm[11:5] | imm[4:0] | rs1   | 000   | rd       | 0000011 | lb     |
| imm[11:5] | imm[4:0] | rs1   | 001   | rd       | 0000011 | lh     |
| imm[11:5] | imm[4:0] | rs1   | 010   | rd       | 0000011 | lw     |
| imm[11:5] | imm[4:0] | rs1   | 100   | rd       | 0000011 | lbu    |
| imm[11:5] | imm[4:0] | rs1   | 101   | rd       | 0000011 | lhu    |
| imm[11:5] | rs2      | rs1   | 000   | imm[4:0] | 0100011 | sb     |
| imm[11:5] | rs2      | rs1   | 001   | imm[4:0] | 0100011 | sh     |
| imm[11:5] | rs2      | rs1   | 010   | imm[4:0] | 0100011 | sw     |

`l` and `s` mean "load" and "store," as mentioned previously.
`b` means a "byte" (8 bits), while `h` means "half" of a word (16 bits).
`u` means "unsigned."

The instructions have the following effects.
`sext(x)` stands for "sign-extend x."
As in C and C++, `&` stands for bit-wise AND.

**Hint**: The data memory port has `mask` and `sext` (sign extend) inputs.
You do not need to mask or sign extend the result outside of the data memory port.

```
lb:  R[rd] = sext(M[R[rs1] + immediate] & 0xff)
lh:  R[rd] = sext(M[R[rs1] + immediate] & 0xffff)
lw:  R[rd] = M[R[rs1] + immediate]
lbu: R[rd] = M[R[rs1] + immediate] & 0xff
lhu: R[rd] = M[R[rs1] + immediate] & 0xffff
sw:  M[R[rs1] + immediate] = R[rs2]
sb:  M[R[rs1] + immediate] = R[rs2] & 0xff
sh:  M[R[rs1] + immediate] = R[rs2] & 0xffff
```

## Testing the other memory instructions

You can run the tests for this part with the following command:

```
sbt:dinocpu> testOnly dinocpu.SingleCycleLoadStoreTesterLab2
```

# Part VII: Branch instructions

This part is a little more involved than the previous instructions.
First, you will implement the branch control unit.
Then, you will wire up the branch control unit and the other necessary MUXes.

## Branch instruction details

The following table show how the branch instructions are laid out.

| imm[12, 10:5] | rs2   | rs1   | funct3 | imm[4:1, 11] | opcode  | Name   |
|---------------|-------|-------|--------|--------------|---------|--------|
| 31-25         | 24-20 | 19-15 | 14-12  | 11-7         | 6-0     |        |
| imm[12, 10:5] | rs2   | rs1   | 000    | imm[4:1, 11] | 1100011 |  beq   |
| imm[12, 10:5] | rs2   | rs1   | 001    | imm[4:1, 11] | 1100011 |  bne   |
| imm[12, 10:5] | rs2   | rs1   | 100    | imm[4:1, 11] | 1100011 |  blt   |
| imm[12, 10:5] | rs2   | rs1   | 101    | imm[4:1, 11] | 1100011 |  bge   |
| imm[12, 10:5] | rs2   | rs1   | 110    | imm[4:1, 11] | 1100011 |  bltu  |
| imm[12, 10:5] | rs2   | rs1   | 111    | imm[4:1, 11] | 1100011 |  bgeu  |

`b` here stands for branch.
`u` again means "unsigned."
The other portion of the mnemonics stand for the operation, either:

* `eq` for equals
* `ne` for not equals
* `lt` for less than
* `ge` for greater than or equal to

The instructions have the following effects.
The operation is given by `funct3` (see above).

```
if (R[rs1] <op> R[rs2])
  pc = pc + immediate
else
  pc = pc + 4
```

## Branch control unit

In this part you will be implementing the branch control component in the CPU design.
The branch control controls whether or not branches are taken.

It takes four inputs: `branch`, `funct3`, `inputx`, and `inputy`.
It will then generate one output, `taken`.

```
branch: true if we are looking at a branch
funct3: the middle three bits of the instruction (12-14). Specifies the type of branch. See RISC-V spec for details.
inputx: first value (e.g., reg1)
inputy: second value (e.g., reg2)
taken:  true if the branch is taken.
```

Note that this is one of the main places the DINO CPU differs from the CPU implemented in the book.
Instead of using the ALU to compute whether the branch is taken or not (the zero output), we are using a dedicated branch control unit.

You must take the RISC-V ISA specification and implement the proper control to choose the right type of branch test.
You must also correctly set or reset the `taken` output if the branch test passes or fails, respectively.
You can find the specification in the following places:

* [the table above](#branch-instruction-details), copied from the RISC-V User-level ISA Specification v2.2, page 104
* Chapter 2 of the Specification
* Chapter 2 of the RISC-V reader
* in the front of the Computer Organization and Design book

Given these inputs, you must generate the correct output on the `taken` wire.
The template code from `src/main/scala/components/branch-control.scala` is shown below.
You will fill in where it says *Your code goes here*.

```
// Control logic for whether branches are taken or not

package dinocpu

import chisel3._
import chisel3.util._

/**
 * Controls whether or not branches are taken.
 *
 * Input:  branch, true if we are looking at a branch
 * Input:  funct3, the middle three bits of the instruction (12-14). Specifies the
 *         type of branch
 * Input:  inputx, first value (e.g., reg1)
 * Input:  inputy, second value (e.g., reg2)
 * Output: taken, true if the branch is taken.
 */
class BranchControl extends Module {
  val io = IO(new Bundle {
    val branch = Input(Bool())
    val funct3 = Input(UInt(3.W))
    val inputx = Input(UInt(32.W))
    val inputy = Input(UInt(32.W))

    val taken  = Output(Bool())
  })

  // Your code goes here.

  io.taken := false.B
}
```

See [the Chisel getting started guide](../chisel-notes/getting-started.md) for examples.
You may also find the [Chisel cheat sheet](https://chisel.eecs.berkeley.edu/2.2.0/chisel-cheatsheet.pdf) helpful.

**HINT:** Use Chisel's `switch` / `is`,  `when` / `elsewhen` / `otherwise`, or `MuxCase` syntax.
You can also use normal operators, such as `<`, `>`, `===`, `=/=`, etc.

### Testing your branch control unit

We have implemented some tests for your branch control unit. The tests, along with the other lab 2 tests, are in `src/test/scala/labs/Lab2Test.scala`.

In this part of the assignment, you only need to run the branch control unit tests.
To run just these tests, you can use the sbt command `testOnly`, as demonstrated below.

```
dinocpu:sbt> testOnly dinocpu.BranchControlTesterLab2
```

## Implementing branch instructions

Next, you need to wire the branch control unit into the data path.
You can follow the diagram given in [the single cycle CPU design section](#single-cycle-cpu-design).
Note that the diagram does not specify what to do with the `taken` result from the branch control unit.
You must add the required logic to drive the correct MUX output based on this `taken` output.

## Testing the branch instructions

You can run the tests for the branch instructions with the following command.

```
sbt:dinocpu> testOnly dinocpu.SingleCycleBranchTesterLab2
```

If you want to test the control unit, use the command [above](#testing-your-branch-control-unit).

# Part VIII: `jal`

Next, we look at the J-type instructions.
You can think of them as "unconditional branches."

## `jal` instruction details

The following table shows how the `jal` instruction is laid out.

| 31-12                    | 11-7 | 6-0     | Name   |
|--------------------------|------|---------|--------|
| imm[20, 10:1, 11, 19:12] | rd   | 1101111 | jal    |

`jal` stands for "jump and link."
The instruction has the following effect.

```
pc = pc + imm
R[rd] = pc + 4
```

## Testing `jal`

You can run the tests for this part with the following command:

```
sbt:dinocpu> testOnly dinocpu.SingleCycleJALTesterLab2
```

# Part IX: `jalr`

`jalr` is very similar to `jal`, with one difference.
However, unlike `jal`, `jalr` has the format of an I-type instruction.

## `jalr` instruction details

The following table shows how the `jalr` instruction is laid out.

| 31-20     | 19-15 | 14-12 | 11-7 | 6-0     | Name   |
|-----------|-------|-------|------|---------|--------|
| imm[11:0] | rs1   | 000   | rd   | 1100111 | jalr   |

`jalr` stands for "jump and link register."
The instruction has the following effect.
(Careful, there's one major difference between this and `jal`!)

```
pc = R[rs1] + imm
R[rd] = pc + 4
```

## Testing `jalr`

You can run the tests for this part with the following command:

```
sbt:dinocpu> testOnly dinocpu.SingleCycleJALRTesterLab2
```

# Part X: Full applications

At this point, you should have a fully implemented RISC-V CPU!
In this final part of the assignment, you will run some full RISC-V applications.

We have provided four applications for you.

* `fibonacci`, which computes the nth Fibonacci number. The initial value of `t1` contains the Fibonacci number to compute, and after computing, the value is found in `t0`.
* `naturalsum`
* `multiplier`
* `divider`

If you have passed all of the above tests, your CPU should execute these applications with no issues!
If you do not pass a test, you may need to dig into the debug output of the test.

## Testing full applications

You can run all of the applications at once with the following test.

```
sbt:dinocpu> testOnly dinocpu.SingleCycleApplicationsTesterLab2
```

To run a single application, you can use the following command:

```
sbt:dinocpu> testOnly dinocpu.SingleCycleApplicationsTesterLab2 -- -z <binary name>
```

# Feedback

This time, instead of uploading a paper version to Gradescope, you will give feedback via a [Google form](https://goo.gl/forms/gNtK47jGKpzNyG252).
Note that the assignment will be out of 90 points and the last 10 points from the feedback will appear in Canvas sometime later.

[Feedback form link](https://goo.gl/forms/gNtK47jGKpzNyG252).

# Grading

Grading will be done automatically on Gradescope.
See [the Submission section](#Submission) for more information on how to submit to Gradescope.

| Name                  | Percentage                 |
|-----------------------|----------------------------|
| Each instruction type | 9% each (× 9 parts = 81%)  |
| Full programs         | 9%                         |
| Feedback              | 10%                        |

# Submission

**Warning**: read the submission instructions carefully.
Failure to adhere to the instructions will result in a loss of points.

## Code portion

You will upload the three files that you changed to Gradescope on the [Lab 2](https://www.gradescope.com/courses/35106/assignments/) assignment.

- `src/main/scala/components/branchcontrol.scala`
- `src/main/scala/components/control.scala`
- `src/main/scala/single-cycle/cpu.scala`

Once uploaded, Gradescope will automatically download and run your code.
This should take less than 5 minutes.
For each part of the assignment, you will receive a grade.
If all of your tests are passing locally, they should also pass on Gradescope unless you made changes to the I/O, **which you are not allowed to do**.

Note: There is no partial credit on Gradescope.
Each part is all or nothing.
Either the test passes or it fails.

## Feedback form

[Give your feedback by filling out the Google form.](https://goo.gl/forms/gNtK47jGKpzNyG252)

Note: You must be logged into Google with your UC Davis email for the link to work.
We will use your email to track whether you have completed the feedback for you to receive your 10 points.

## Academic misconduct reminder

You are to work on this project **individually**.
You may discuss *high level concepts* with one another (e.g., talking about the diagram), but all work must be completed on your own.

**Remember, DO NOT POST YOUR CODE PUBLICLY ON GITHUB!**
Any code found on GitHub that is not the base template you are given will be reported to SJA.
If you want to sidestep this problem entirely, don't create a public fork and instead create a private repository to store your work.
GitHub now allows everybody to create unlimited private repositories for up to three collaborators, and you shouldn't have *any* collaborators for your code in this class.

## Checklist

- [ ] You have commented out or removed any extra debug statements.
- [ ] You have uploaded three files: `cpu.scala`, `control.scala`, and `branchcontrol.scala`.
- [ ] You have filled out the [feedback form](https://goo.gl/forms/gNtK47jGKpzNyG252).

# Hints

- Start early! There is a steep learning curve for Chisel, so start early and ask questions on Piazza and in discussion.
- If you need help, come to office hours for the TAs, or post your questions on Piazza.

## `printf` debugging

This is the best style of debugging for this assignment.

- Use `printf` when you want to print *during the simulation*.
  - Note: this will print *at the end of the cycle* so you'll see the values on the wires after the cycle has passed.
  - Use `printf(p"This is my text with a $var\n")` to print Chisel variables. Notice the "p" before the quote!
  - You can also put any Scala statement in the print statement (e.g., `printf(p"Output: ${io.output})`).
  - Use `println` to print during compilation in the Chisel code or during test execution in the test code. This is mostly like Java's `println`.
  - If you want to use Scala variables in the print statement, prepend the statement with an 's'. For example, `println(s"This is my cool variable: $variable")` or `println("Some math: 5 + 5 = ${5+5}")`.

## Common errors

See the [first lab](../lab1/lab1.md#common-errors) for more common errors.
We'll add common errors specific to this lab as we see them on Piazza.
