# The FPP-8/X architecture
# NOT YET COMPLETE

The FPP-8/X is a rethink of the DEC FPP-8, a floating point
coprocessor meant to work with the PDP-8/X.
Floats are stored in a single 32-bit word.

Like PDP-8/X instructions, FPP-8/X instructions are also
32 bits long, although only the least significant 28 bits are
used.

[link to documentation for the FPP-8 instruction set]

Because the FPP-8/X architecture is designed as a deterministic
float machine, there is no support for integer arithmetic or
for interrupts.  The FPP can cooperate closely with the CPU,
because it shares the same memory, though not the same registers.

It should be fairly easy to implement this architecture on a chip;d.

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
    
  * The 5-bit FOP (operation) register
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
pointed to by FBASE.  There are also 15 indexed pages,
whose beginnings are pointed to by the 15 words following
where the FX0 register points (the word pointed to by FX0 itself
is not available).  Neither the base page nor the indexed pages
have to begin at a multiple of address 0x0000_0800.
   

We write M[x] to represent the contents of the word in memory
whose address is x.  We write F[x] to represent M[x] when
interpreted as a float.

## Instruction decoding

Instructions are fetched, decoded, and executed as follows:

 * Set FIR to M[FPC].
   
 * Set FINC, FOP, FOPX, FIDX, and FOFF
   to the 0x0002_0000, 0x0001_F000, 0x00F0_0000,
   0x0F00_0000, and 0x0000_07FF
   bits of FIR respectively.
   
 * Set the 21 most significant bits of FY
   to the 21 most significant bits of FPC,
   and the 11 least significant bits of FY to 0.
   
 * Set FPC to FPC + 1, ignoring any overflow.
   
 * If FPC is greater than H, set PC to 0.
      
## Memory references

Unlike the PDP-8/X, the FPP-8/X has very few
instructions that do not reference memory, so
the following steps are taken for most instructions.
An implementation can skip them when executing any of
the instructions that do not make use of the FY register.

The first step is to determine an initial value of FY,
using the page bit of FIR (the 0x0000_0800 bit) and
the FBASE, FIDX, and FOFF registers.

If the page bit is 0, then set FY to FBASE + FOFF,
so that Y represents an address on the base page.

If the page bit is 1 and FIDX is 0, then set FY to FY + FOFF,
so that FY represents an address on the current page.

If the page bit is 1 and FIDX is not 0,
then check FINC, and if it is 1, increment
M[M[FX0] + FIDX] by 1, ignoring any overflow.
Then set FY to M[M[FX0]+FIDX] + FOFF,
so that FY represents an address in one of the
15 pages addressable by the in-memory index registers.

If the 0x08 bit of FOP is 1, then set FY to M[FY].
In addition, if FY = FPC, set FPC to FPC + 1, thus skipping
M[FY] if it immediately follows the instruction.
This is called *indirect addressing*.

If the 0x08 bit of FOP is 0 it is called
*direct addressing*, and no special action is taken.

Any word in memory can be
accessed indirectly, but only the 2KW of the base page
the 15 index-accessible pages of 2KW each, and the 2KW of the current page
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
   
 * If FOP is 0x12, then check FINC.
   If it is 1, then increment M[M[FX0] + FIDX] by 1.
   In any case,  if M[M[FX0] + FIDX] is 0 then set PC to Y.
 
 * If FOP is 0x13 through 0x17,
   then set the 0x8 bit of FST to 1,
   and break out of the instruction execution loop.
 
## Special instructions where FOP = 0x10

## Special instructions where FOP = 0x11
 
