**Note: The architecture is being redesigned to use 32-bit instructions.  This paragraph will be removed when everything has been updated.**

# The PDP-8/X architecture

The PDP-8/X is a rethink of the DEC PDP-8, a 12-bit minicomputer.
Because the PDP-8 is word-oriented, it almost doesn't matter
how big the words are, so we extend them from 12 bits to 32 bits.
Good resources for the final PDP-8 models, the PDP-8/E, PDP-8/F, PDP/8-M, and
PDP-8/A (the letters are meaningless and not even alphabetical), are the
*[PDP-8/E and PDP/8-M Small Computer Handbook 1973](http://www.vandermark.ch/pdp8/uploads/PDP8/PDP8.Manuals/DEC-S8-OSSCH-A.pdf)*
(note that there are many editions of the *Small Computer Handbook*,
each significantly different from the previous one) and the
*[PDP/8-A Miniprocessor Users Manual 1976](http://www.bitsavers.org/pdf/dec/pdp8/pdp8a/EK-8A002-MM-002_PDP-8A_Miniprocessor_Users_Manual_Dec76.pdf)*.
This document is much less comprehensive, and will attempt to explain
as clearly as possible what a PDP-8/X CPU actually appears to do.
(Of course the implementation may be quite different, provided the
user-visible registers, memory locations, and behaviors are the same.)

Because the PDP-8/X architecture is designed as a deterministic
integer machine, there is currently no support for interrupts, which on the
PDP-8 are signaled by simulating a JMS instruction to address 0.

It should be fairly easy to implement this architecture on a chip;
the ordinary PDP-8 was eventually made into a chip,
though most of the 50,000 PDP-8s built were not chip-based.

## Representation

The 32-bit words can be interpreted as
either a vector of 32 bits from the most significant bit
to the least significant bit, or a signed integer value
between -2_147_483_648 and 2_147_483_647 inclusive.
(Underscores are used to make numbers more readable.)
We can also interpret a word as two halfwords,
the 16 most significant bits
(the most significant halfword),
and the 16 least significant bits
(the least significant halfword).

The representation of the integer is twos-complement,
like most machines today. That is, a word is negated
by bitwise-NOTting it and then adding 1 to the result.

We write numeric values either as signed decimal numbers
or as unsigned hexadecimal values starting with 0x.
This is different from PDP-8 architecture descriptions,
which use octal numbers, grouping the 12 bits as 4 groups of 3. 

We can also write them as decimal numbers with a decimal point or exponent or both, in which case they they are tepresented ss IEEE float32 values

In order to talk about individual bits,
we speak of the 0x8000_0000 bit (most significant and the sign bit),
the 0x4000_0000 bit (slightly less
significant), and so on down to the 0x0000_0001 bit (least significant).
We can use similar bit masks to talk about multiple bits at the same time.

## Registers

The user-visible registers of a PDP-8/X are very few by modern standards:

 * The 32-bit AC or accumulator register, used
   by almost all instructions.
        
 * The 32-bit MQ or multiplier-quotient register.
     
  * The 1-bit L or link register.  When adding a value to
    the AC, any overflow changes the L register from 0 to 1
    or from 1 to 0.
     
  * The 32-bit PC or program counter points to the next memory location to be
    executed. Its value is an unsigned integer
    between 0 and however much memory is installed in the system.
     
 * The 32-bit BASE register holds a memory address
    of which the low-order 11 bits must be 0.

 * The 32-bit SBASE register holds a memory address
   of which the low-order 11 bits must be 0.

  The following registers are not visible to programmers and are used
  in this explanation; they may or may not correspond to actual registers:
  
  * The 32-bit R register holds the instruction word being executed.
      
  * The 5-bit OP register contains the 0x7D_0000 bits of IR.
  
  * The 1-bit S register is 1 if the next instruction is going to be skipped
    (not executed) and 0 otherwise.

  * The -bit SP register 
    
  * The 32-bit Y register contains the address of the memory location 
    being accessed by the current instruction (if any).
     
   * The 8-bit D register contains the number of an I/O device.
  
   * The 4-bit DOP register contains the number of
     a device-specific I/O instruction.
          
   * The 7-bit SC (shift count) register holds the number of shifts to be
performed by an extended arithmetic shift instruction.
   
   * The 32-bit H register is set when the machine is built,
     and contains the address of the last existing word in memory.
     
   * The 1-bit B register is used to save a bit while rotating.
           
## Memory
   
Memory is measured in 32-bit words.
It is organized into *pages* that are 2 KW in length.
This use of the term *page* has nothing to do
with virtual memory.

The page whose addresses are #x0000_0000 to #x0000_07FF inclusive is called
*page zero*.  The page that contains the instruction currently being executed is
called the *current page*.  The page whose lowest address is the BASE register is called the *base page*.  Finally, the page whose highest address is in the SBASE register is called the *stack page*.
   
The maximum amount is 512K pages, 1 GW; in a fully equipped
machine the address of the last word is 0x3FFF_FFFF.  A read from
a non-existent page returns 0; a write to a non-existent page does nothing.

(To determine if a page exists at run time, read a word from the page.
If the value is not 0, the page exists.  Otherwise, write a non-zero value
to the same word and read it a second time.  If the value is 0, the page
does not exist.  Otherwise the page exists; write 0 to the word.)

We write M[x] to represent the contents of the word in memory
whose address is x.

## Instruction decoding

Instructions are fetched, decoded, and executed as follows.

 * Start by setting IW to M[PC].

 * Set OP to the 0x bits of IR.
   
 * Set PC to PC + 1, ignoring any overflow.
   
 * If PC is greater than H, set PC to 0.

 * If S is 0, execute the instruction based on the contents of OP
   and the details in the following sections, which are mutually
   exclusive.
   
 * Set S to 0.

Then repeat from the beginning.
   
## Immediate value instruction

If the 0x8000_0000 bit of IR is 1, set the 0x8000_0000 bit of
IW to the 0x4000_0000 bit of IR.  Then set AC to IR.
The assembler mnemonic is IMM.

## Memory-referencing instructions
   
If the value of OP is anything except 0x06 or 0x07,
IR contains a memory-referencing instruction (MRI), all of which have the same format.
The Y register is set in a common way for all MRIs, and then the value of OP specifies
exactly what to do with Y, memory locations, and the user-visible registers of the PDP-8/X.

The first step is to determine an initial value of Y using the page bits of IR, which is
the 0x0800 bit, and the 11 least significant bits of the IR, which are the 0x07FF bits.
If the page bit is 0, then the 21 most significant bits of Y are set to 0, so that
Y represents an address on page zero.  If the page bit is 1, Y will represent an address
on the current page, and no special action needs to be taken.

The 11 least significant bits of IR are
copied to the 11 least significant bits of Y.

If the indirect bit (that is, the 0x1000 bit) of IR is 1,
if Y is in the range 0000_0200 to 0000_02FF inclusive,
set M[Y] to M[Y] + 1.  In any case set Y to M[Y].
This is called *indirect addressing*.

If the indirect bit is 0, nothing is done.
This is called *direct addressing*

Any word in memory can be
accessed indirectly, but only the 2KW of page zero and the 2KW of the current page
can be accessed directly.

Then one of the following six cases is chosen:

 * If OP is 0x0, then set AC to AC bitwise-ANDed with M[Y].
   The assembler mnemonic is AND.
 
 * If OP is 0x1, then set AC to the sum of AC and M[Y].  If
   there is an overflow out of AC as a result, then set L to 1 - L.
   The assembler mnemonic is, for historical reasons, TAD (two's
   complement add).
 
 * If OP is 0x2, then set M[Y] to AC, and then set AC to 0.
   The assembler mnemonic is DCA (deposit and clear AC).
 
 * If OP is 0x3, then set M[Y] to M[Y] + 1; any overflow is ignored.
   If M[Y] is now 0, then set S to 1.
   If PC is now greater than H, then set PC to 0.
   The assembler mnemonic is ISZ (increment and skip if zero).
 
 * If OP is 0x4, then set M[Y] to PC + 1
   and set PC to Y + 1.
   (Note that if this instruction is in the least significant bits
   of a memory location,
   the instruction in the most significant bits will never be executed.)
   The assembler mnemonic is JMS (jump to subroutine).
 
 * If OP is 0x5, then set PC to Y.
   (Note that if this instruction is in the least significant bits,
   the instruction in the most significant bits of a memory location
   will never be executed.)
   The assembler mnemonic is JMP.
 
## I/O instructions
 
If OP is 0x6, then set D to the 0x0FF0 bits of IR
and set DOP to the 0x000F bits of IR.
The effect of executing such an instruction depends on
the values of D and DOP, and is documented
[elsewhere](devs.md).
If D corresponds to an unknown or unavailable device, or
DOP to an undefined instruction, then no action is taken.
 
## Operate instructions
 
If OP is 0x7 and the 0x0800 bit is 0,
then various operations on AC, L, and/or MQ are
performed depending on the bits of IR,
which are examined in the order given below.
Note that all applicable actions are taken, not just the first one.
 
For convenience, the 0x0010 bit is called the RR (rotate right) bit,
the 0x0004 bit is called the RL (rotate left) bit, 
and the 0x0002 bit is called the RT (rotate twice) bit.

 * If the 0x0200 bit of IR is 1, then set AC to 0.
   The assembler mnemonic is CLA (clear AC).
  
 * If the 0x0100 bit of IR is 1, then set L to 0.
   The assembler mnemonic is CLL (clear link).

 * If the 0x0040 bit of IR is 1, then set AC to the bitwise-NOT of AC,
   or equivalently to (-A) - 1.
   The assembler mnemonic is CMA (complement AC).
   
 * If the 0x0020 bit of IR is 1, then set L to 1 - L.
   The assembler mnemonic is CML (clear link).
 
 * If the 0x0001 bit of IR is 1, then set AC to AC + 1.
   If there is an overflow out of AC as a result, set L to 1 - L.
   The assembler mnemonic is IAC (increment AC).
 
 * If the RR bit of IR is 1 and the RL and RT bits of IR are 0,
   then jointly rotate AC and L right by one bit.
   That is, set B to the lowest order bit of AC, shift AC right by one bit,
   set the most significant bit of AC to L
   and set L to B.
   The assembler mnemonic is RAR (rotate AC right).
   
 * If the RR and RT bits of IR are 1 and the RL bit of IR is 0,
   then jointly rotate AC and L right by two bits.
   This can be done by rotating right by one bit twice or more efficiently.
   The assembler mnemonic is RTR (rotate twice right).
   
 * If the RL bit of IR is 1 and the RR and RT bits of IR are 0,
   then jointly rotate AC and L right by one bit.
   That is, set B to L, shift AC left by one bit,
   and set the least significant bit of AC to L.
   The assembler mnemonic is RAL (rotate AC left).
   
 * If the RL and RT bits of IR are 1 and then RR bit of IR is 0,
   then jointly rotate AC and L left by two bits.
   This can be done by rotating left by one bit twice or more efficiently.
   The assembler mnemonic is RTL (rotate twice left).
   
 * If the RL and RR bits are 0 and the RT bit is 1, then exchange the
   16 most significant bits of AC with the 16
   16 least significant bits  of AC.
   The assembler mnemonic is HSW (halfword swap); there is no PDP-8 equivalent.
   
 * If the RL, RR, and RT bits are all 1, then set AC to PC.
   The assembler mnemonic is PCA (PC to AC); this is a PDP/8-A-only
   instruction without its own mnemonic.
 
 * If the RL and RR bits are 1 and the RT bit is 0, then exchange the
   0xF000 and the 0x0F00 bits of AC and exchange
   0x00F0 and the 0x000F bits of AC.
   The assembler mnemonic is BSW (byte swap); the PDP-8 equivalent
   swaps the two six-bit halfwords and so is more like the HSW instruction.
   
Note that if all the 0x1FFF bits are zero, no action is taken.
The assembler mnemonic is NOP.
    
## Skip instructions
 
If OP is 0x7 and the 0x0800 bit of IR is 1
but the 0x0001 bit of IR is 0,
then S is set depending on various bits
of IR, AC, and L.
The bits of IR are examined in the order given below.
Note that *all* applicable actions are taken, not just the first one.

 * If the 0x0200 bit of IR is 1, then set AC to 0.
   There is no assembler mnemonic.
 
 * If the 0x0100 bit of IR = 1 and the sign (0x80000000) bit of AC = 1,
   (that is, AC is negative), then set S to 1.
   The assembler mnemonic is SMA (skip if minus AC).
  
 * If the 0x0040 bit of IR = 1 and AC = 0, then set S to 1.
    The assembler mnemonic is SZA (skip if zero AC).
 
 * If the 0x0020 bit of IR = 1 and L = 1, then set S to 1.
   The assembler mnemonic is SNL (skip if non-zero L).
 
 * If the 0x0010 bit of IR = 1, then set S to 1 - S.
   The assembler mnemonics for the previous three instructions
   when combined with this bit are SPA (skip if positive-or-zero AC),
   SNA (skip if non-zero AC), and SZL (skip if zero L) respectively.
  
 * If the 0x0002 bit of IR = 1, then halt the PDP-8/X processor.
   When the processor is restarted externally, all registers
   (user-visible and not) and all of memory are unchanged.
   The assembler mnemonic is HLT.

## AC and MQ instructions

If OP is 0x7 and the 0x0800 bit and the 0x0001 bit of IR are both 1,
then the bits of IR are examined in the order given below.
Note that *all* applicable actions are taken, not just the first one.

 * If the 0x0200 bit is 1, then set AC to 0.
   There is no assembler mnemonic.
   
 * If the 0x0400 bit is 1, then set AC to AC bitwise-ORed with MQ.
   The assembler mnemonic is MQA (MQ to AC).
 
 * If the 0x0100 bit is 1, then set MQ to AC.
   The assembler mnemonic is MQL (MQ Load).

 * If the 0x0400 and the 0x0100 bits are both 1, then exchange MQ and AC.
   The assembler mnemonic is SWP.
   
## Extended arithmetic instructions

When an AC and MQ instruction is complete,
the EOP register is set to the 0x00F0 bits of IR
and one of the following operations is executed:

 * If EOP is 0x0, no action is taken.

 * If EOP is 0x1, set SC is the 11 low-order bits of AC,
   and set AC to 0.
   The assembler mnemonic is ASC (AC to SC).
   (Note that some DEC documentation claims the mnemonic
   is the meaningless ACS.)
 * If EOP is 0x2, set Y to M[PC] and PC to PC + 1.
   If the 0x1000 (indirect) bit of IR is 1, set Y to M[Y].
   The sum of AC and the 64-bit unsigned product of Y and MQ is computed.
   Set AC to the 32 high-order bits of the product,
   and MQ to the 32-low-order bits.  Set L and SC to 0.
   The assembler mnemonic is MUY (Multiply).
     
 * If EOP is 0x3, set Y to M[PC] and PC to PC + 1.
   If the 0x1000 (indirect) bit of IR is 1, set Y to M[Y].
   The 64-bit unsigned number whose 32 high-order bits are
   in the AC and whose 32 low-order bits are in the MQ is
   divided by Y.  Set MQ to the quotient and AC to the remainder.
   Set L to 1 if there is a divide overflow and set to 0 if not.
   Set SC to 0.
   The assembler mnemonic is DVI (Divide).
     
 * If EOP is 0x4, SC is set to 0.
   Then L, AC, and MQ are shifted left as described
   for the LSH instruction.
   The left shift is repeated, setting SC to SC + 1, until
   either the 0x8000_0000 and 0x4000_0000 bits of AC are different,
   or AC is 0xC000_0000 and MQ is 0.
   The assembler mnemonic is NMI (Normalize)
      
 * If EOP is 0x5, the L, AC, and MQ are treated as a 65-bit register,
   where L is the high-order bit, AC is the 32 middle bits, and MQ is
   the 32 low-order bits.  To shift:
   
   * set L to the 0x8000_0000 bit of AC
   * shift the bits of AC left
   * set the 0x0000_0001 bit of AC to the 0x8000_0000 bit of MQ
   * shift the bits of MQ left
   * set the 0x0000_0001 bit of MQ to zero.

   The SC register specifies how many shifts to perform.
   At the end, set SC to 0.
   The assembler mnemonic is SHL (Shift Left).
     
 * If EOP is 0x6, the L, AC, and MQ are treated as a 65-bit register,
   where L is the high-order bit, AC is the 32 middle bits, and MQ is
   the 32 low-order bits.  To shift:
   
   * set L to 0
   * shift the bits of MQ right
   * set the 0x8000_0000 bit of MQ to the 0x0000_0001 bit of AC
   * shift the bits of AC right
   * set the 0x8000_0000 bit of AC to L
  
   The SC register specifies how many shifts to perform.
   At the end, SC is set to 0.
   The assembler mnemonic is LSR (Logical Shift Right).
   
 * If EOP is 0x7, the L, AC, and MQ are treated as a 65-bit register,
   where L is the high-order bit, AC is the 32 middle bits, and MQ is
   the 32 low-order bits.  To shift:
   
   * set L to 0
   * shift the bits of MQ right
   * set the 0x8000_0000 bit of MQ to the 0x0000_0001 bit of AC
   * shift the bits of AC right
   * set the 0x8000_0000 bit of AC to L
  
   The SC register specifies how many shifts to perform.
   At the end, SC is set to 0.
   The assembler mnemonic is ASR (Arithmetic Shift Right).
   
 * If EOP is 0x8, set AC to AC bitwise-ORed with SC.
   The assembler mnemonic is SCA (Step Counter to AC).
   
 * If EOP is 0x9, set Y to M[PC] and PC to PC + 1.
   If the 0x1000 (indirect) bit of IR is 1, set Y to M[Y].
   Then set MQ to the sum of MQ and M[Y+1], and set AC
   to the sum of AC, M[Y], and the carry from MQ.
   Set L to the carry from AC.
   The assembler mnemonic is DAD (Double Add).
   
 * If EOP is 0xA, set M[Y] to AC and M[Y+1] to MQ.
   The assembler mnemonic is DST (Double Store).
   
 * If EOP is 0xB, take no action.
   
 * If EOP is 0xC, set MQ to the sum of MQ and 1.
   Then set AC to the sum of AC and the carry from MQ.
   Set L to the carry from AC.
   The assembler mnemonic is DPIC (Double Precision Increment).
   
 * If EOP is 0xD, set S to 1 if both AC and MQ are 0.
   Otherwise set S to 0.
   The assembler mnemonic is DPSZ (Double Precision Skip if Zero).
 
 * If EOP is 0xE, set AC to the bitwise-NOT of AC
   and MQ to the bitwise-NOT of MQ.
   The assembler mnemonic is DCM (Double Complement).
  
 * If EOP is 0xF, set AC to MQ - AC.  Set L to 1 if
   a borrow is propagated from the most significant bit of AC.
   Otherwise, set L to 0.
   The assembler mnemonic is SAM (Subtract AC from MQ).

## Ideas

MQ operates and skips
```
muurkha> did I already suggest having a couple of extra page selector registers and an extra page selection bit in the instruction word?
4:21 PM <muurkha> you could use one page selection register for a stack and another for a self pointer
4:25 PM <muurkha> I mean, not as a mandatory thing, just that some programs might choose to use them that way
4:26 PM <muurkha> the idea is that instead of a single zero-page/pc-page selection bit, you have a two-bit field where maybe 00 is zero-page, 01 is pc-page, 10 is ss, and 11 is rs
4:26 PM <muurkha> and instructions to load and store ss and rs
4:27 PM <muurkha> I feel like the indirection bit kind of makes PDP-8 halfway CISC already
4:36 PM <jcowan> there's really no alternative to the indirect bit
4:36 PM <jcowan> let me see if I understand this
4:38 PM <jcowan> okay, now we have 18 bits [no, 17 bits]
4:39 PM <jcowan> The 0xB instruction in the EAE range is still free
4:40 PM <jcowan> I note that SAM says it manipulate the Greater Than Flag, so I have to add that (and maybe use 0B in the EAE range, which currently is undefined, to get/set it
4:43 PM <jcowan> The X86 segment registers number four because Pascal has four memory spaces code, globals, stack, heap and you statically know for nay pointer which one it is.
4:43 PM <jcowan> s/nay/any
4:46 PM <jcowan> so maybe we leave the indirect bit alone and add 2 more bits for these spaces
```

PDP-11 instructions:
 * decrement AC
 * add L to AC
 * subtract L from AC
 * xor, bit set, bit clear
 * decrement and skip if zero
 * JSR, RTS

FPP-8/A instructions:
 * FLDA, FADD, FSUB, FMUL, FDIV, FADDM, FMULM, FSTA
 * FINT, FFLT
