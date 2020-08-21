# The FPP-8/X architecture

The FPP-8/X is a rethink of the DEC FPP-8, a floating point
coprocessor meant to work with the PDP-8/X.
Floats are stored in a single 32-bit word

Like PDP-8/X instructions, FPP-8/X instructions are also
32 bits long, although only the least significant 28 bits are
used.

[link to documentation for the FPP-8 instruction set]

Because the FPP-8/X architecture is designed as a deterministic
float machine, there is no support for integer arithmetic or
for interrupts.  The FPP can cooperate closely with the CPU,
because it shares the same memory, though not the same registers.

It should be fairly easy to implement this architecture on a chip.

## Issues

There needs to be a way to stop the FPP when the CPU must service
a device without using interrupts.
The best idea so far is to have a line on the bus that a device
signals if it is done and which the FPP monitors, forcing an FPHLT.

## Representation

The 32-bit words can be interpreted in the same
way as PDP-8/X words, but also as
IEEE 754(2008) 32-bit binary floats.
The question of endianism is still open.

We write float values either as signed decimal numbers
(with either a decimal point or an exponent) or as
the values +inf.0, -inf.0, and +nan.0
or as unsigned hexadecimal values starting with 0x.

## Registers

The user-visible registers of a FPP-8/X are part of its
device nature and are documented elsewhere.

The following registers are not visible to programmers and are used
in this explanation; they may or may not correspond to actual registers:
  
  * The 32-bit FIR register
    contains the instruction currently being executed.
    FIR is always treated as a bit vector.
        
  * The 4-bit FOP (operation) register
    contains the bits of FIR used to specify
    the operation to be done.
  
  * The 4-bit FOPX (extended operation) register
    contains the bits of FIR used to specify the
    operation to be done when FOP is 0x10 or 0x11.
  
  * The 4-bit FIDX (index register number) register
    contains the bits of FIR used to specify the
    index register (if any) used by the current instruction.
    
  * The 11-bit FOFF (offset) register
    contains the bits of FIR used to specify the offset
    from the base register, one of the index registers,
    or the first instruction on the current page.
               
## Memory
   
Memory is measured in 32-bit words.
It is organized into *pages* that are 2 KW in length.
Page zero is not special to the FPP; its role is replaced by the
*base page*, which is a 2KW page whose beginning is
pointed to by FBASE.  The base page does not
have to begin at a multiple of 0x800.

There are also 16 index registers,
which resided in memory starting at the address
where the FX0 register points.

We write M[x] to represent the contents of the word in memory
whose address is x.  We write F[x] to represent M[x] when
interpreted as a float.  We write I[x] to represent the
contents of index register x; it is equivalent to M[FX0+x].

## Conversion

Although the FAC normally contains a float, its value can be
converted to an integer as follows:

 * If FAC is +nan.0, set FAC to 0x0000_0000.
 * If FAC is greater than 2^31-1,
   set FAC to 0x7FFF_FFFF.
 * If FAC is greater than -2^31,
   set FAC to 0x8000_0000
 * Otherwise, truncate the float value of FAC towards zero,
   and set FAC to the numerically equivalent integer value.
   
 Correspondingly, when FAC contains an integer, it can be
 converted to the numerically equivalent float.

## Initializing the FPP-8/X

When the FPP starts running due to a CPU I/O instruction,
the registers are initialized as follows:

* Set FPC to M[FAPT+1]
* Set FX0 to M[FAPT+2]
* Set FBASE to M[FAPT+3]
* Set FY to M[FAPT+4]
* Set FAC to M[FAPT+5]

## Instruction decoding

When the FPP starts running,
instructions are fetched, decoded, and executed as follows:

 * Set FIR to M[FPC].
   
 * Set FIDX, FOPX, FOP, and FOFF registers
   to the 0x0F00_0000, 0x00F0_0000, 0x0001_F000, and 0x0000_07FF 
   bits of FIR respectively.
   
 * Set the 21 most significant bits of FY
   to the 21 most significant bits of FPC,
   and the 11 least significant bits of FY to 0.
   
 * Set FPC to FPC + 1, ignoring any overflow.
   
 * If FPC is greater than H, set PC to 0.
      
## Memory references

Unlike the PDP-8/X, the FPP-8/X has very few
instructions that do not reference memory, so
the following steps are taken for all instructions
except those where FOP = 0x10.

The first step is to determine an initial value of FY,
using the page bit of FIR (the 0x0000_0800 bit) and
the FBASE, FIDX, and FOFF registers.

If the page bit is 0, then set FY to FBASE + FOFF,
so that Y represents an address on the base page.

If the page bit is 1, then set FY to FY + FOFF,
so that FY represents an address on the current page.

If the page bit is 1 and FIDX is not 0,
then check the increment bit of FIR
(the 0x0004_0000 bit) and if it is 1, increment
I[FIDX] by 1, ignoring any overflow.
In either case, set FY to FY + I[FIDX],
so that FY is offset by one of the 16 (in-memory) index registers.

If the indirect bit (the 0x0002_0000 bit) of FIR 
is 1, then set FY to M[FY].
In addition, if FY = FPC, set FPC to FPC + 1 (ignoring any overflow),
thus skipping M[FY] if it immediately follows the instruction.
This is called *indirect addressing*.

If the indirect bit of FOP is 0 it is called
*direct addressing*, and no special action is taken.

Any word in memory can be
accessed indirectly, but only the 2KW of the base page
and the 2KW of the current page, possibly extended by indexing, 
can be accessed directly.

## Ordinary instructions

This section describes how to interpret instructions
other than those whose FOP value is 0x10 or 0x11.

 * If FOP is 0x00 or 0x08, then set FAC to F[FY].
   The assembler mnemonic is FLDA.
 
 * If FOP is 0x01 or 0x09, then set FAC to FAC + F[FY].
   The assembler mnemonic is FADD.
 
 * If FOP is 0x02 or 0x0A, then set FAC to FAC - F[FY].
   The assembler mnemonic is FSUB.
 
 * If FOP is 0x03 or 0x0B, then set FAC to FAC / F[FY].
   The assembler mnemonic is FDIV.
 
 * If FOP is 0x04 or 0x0C, then set FAC to FAC * F[FY].
   The assembler mnemonic is FMUL.
 
 * If FOP is 0x05 or 0x0D, then set F[FY] to FAC + F[FY].
   The assembler mnemonic is FADDM.
 
 * If FOP is 0x06 or 0x0E, then set F[Y] to FAC.
   The assembler mnemonic is FSTA.
 
 * If FOP is 0x07 or 0x0F, then set F[Y] to FAC * F[FY].
   The assembler mnemonic is FMULM.
   
 * If FOP is 0x12, then set PC to Y.
   The assembler mnemonic is JXN.
 
 * If FOP is 0x13 through 0x17,
   then set FST to 0x8, set FF to 1,
   and stop the instruction execution loop.
   The assembler mnemonics are TRAP3 through TRAP7.
   
  By convention, when the CPU gets control
  after a TRAP3, it issues a JMP to FY
  FY and does not automatically restart the FPP.
  In case of a TRAP4, the CPU issues a JMS to FY
  and restarts the FPP after the JMS returns.
  This logic is implemented in the PDP/8-X code
  that waits for the FPP to stop.
   `
## Special instructions where FOP = 0x10

Note that none of the following instructions
depend on FY, which therefore must not be computed.

 * If FOPX is 0x0,
   then set FST to 0x0, FF to 1,
   and stop the instruction execution loop.
   The assembler mnemonic is FEXIT.
   
 * If FOPX is 0x1,
   set FAC to the integer value of FAC.
   The assembler mnemonic is FINT.
 
 * If FOPX iz 0x2,
   set FAC to the integer value of FAC, and then
   set I[FIDX] to FAC.   
   The assembler mnemonic is ATX.
   
 * If FOPX is 0x3,
   set FAC to to I[FIDX], and then
   set FAC to the float value of FAC.
  The assembler mnemonic is XTA.
 
 * If FOPX is 0x4,
   do nothing.
   The assembler mnemonic is FNOP.
 
 * If FOPX is 0x8,
   set I[FIDX] to M[FPC] and
   then set FPC to FPC + 1, ignoring overflow.
   The assembler mnemonic is LDX.
 
 * If FOPX is 0x9,
   set I[FIDX] to the sum
   of I[FIDX] and M[FPC];
   then set FPC to FPC + 1, ignoring overflow.
   The assembler mnemonic is ADDX.
 
 * If FOPX is 0xA,
   set FAC to the float value of FAC.
   The assembler mnemonic is FFLT.
 
Otherwise do nothing.

## Special instructions where FOP = 0x11
 
 * If FOPX is 0x0,
   then if FAC is zero then set FPC to FY;
   otherwise do nothing.
   The assembler mnemonic is JEQ.
 
 * If FOPX is 0x1,
   then if FAC is not negative then set FPC to FY;
   otherwise do nothing.
   The assembler mnemonic is JGE.
 
 * If FOPX is 0x2,
   then if FAC is not positive then set FPC to FY;
   otherwise do nothing.
   The assembler mnemonic is JLE.

 * If FOPX is 0x3,
   then set FPC to FY.
   The assembler mnemonic is JA.

 * If FOPX is 0x4,
   then if FAC is not zero then set FPC to FY;
   otherwise do nothing.
   The assembler mnemonic is JNE.

 * If FOPX is 0x5,
   then if FAC is negative then set FPC to FY;
   otherwise do nothing.
   The assembler mnemonic is JLT.

 * If FOPX is 0x6,
   then if FAC is positive then set FPC to FY;
   otherwise do nothing.
  The assembler mnemonic is JGT.

 * If FOPX is 0x7,
   then if FAC is less than -2^32^ or
   greater than 2^32-1 then set FPC to FY;
   otherwise do nothing.
   The assembler mnemonic is JAL.

 * If FOPX is 0x8,
   set FX0 to FY.
   The assembler mnemonic is SETX.

 * If FOPX is 0x9,
   set FBASE to FY.
   The assembler mnemonic is SETB.

 * If FOPX is 0xA,
   set M[FY] to a JA instruction
   that when executed will jump indirectly via FY + 1,
   set M[FY+1] to FPC, and set FPC to FY + 2, ignoring overflow.
   The assembler mnemonic is JSA.

 * If FOPX is 0xB,
   set M[FY] to FPC
   and then set M[FY+1] to FPC + 1, ignoring overflow.
   Then set FPC to FY + 1, ignoring overflow.
   The assembler mnemonic is JSR.

Otherwise do nothing.

## Stopping the instruction loop

When the FPP stops processing instructions due to a
FEXIT, FPAUSE, FTRAPn, or CPU FPST instruction,
the following memory locations are set:

 * Set M[FAPT+1] to FPC
 * Set M[FAPT+2] to FX0
 * Set M[FAPT+3] to FBASE
 * Set M[FAPT+4] to FPC
 
 Then set FF to 1.

