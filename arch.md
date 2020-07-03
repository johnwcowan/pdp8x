# The PDP-8/X architecture

The PDP-8/X is a rethink of the DEC PDP-8, a 12-bit minicomputer.
Because the PDP-8 is word-oriented, it almost doesn't matter
how big the words are, so we extend them from 12 bits to 32 bits.
However, the instructions are only 16 bits long; to keep the machine
word-oriented, the remaining bits are unused except in one case.

Good resources for the final PDP-8 models, the PDP-8/E, PDP-8/F, PDP/8-M, and
PDP-8/A (the letters are meaningless and not even alphabetical), are the
*[PDP-8/E and PDP/8-M Small Computer Handbook 1973](http://www.vandermark.ch/pdp8/uploads/PDP8/PDP8.Manuals/DEC-S8-OSSCH-A.pdf)*
(note that there are many editions of the *Small Computer Handbook*,
each significantly different from the previous one) and the
*[PDP/8-A Miniprocessor Users Manual 1976](http://www.vandermark.ch/pdp8/uploads/PDP8/PDP8.Manuals/EK-8A002-MM-002.pdf)*.
This document is much less comprehensive, and will attempt to explain
as clearly as possible what a PDP-8/X CPU actually appears to do.
(Of course the implementation may be quite different, provided the
user-visible registers, memory locations, and behaviors are the same.)

Because the PDP-8/X architecture is designed as a deterministic
integer machine, there is no support for interrupts, which on the
PDP-8 are signaled by simulating a JMS instruction to address 0.
The optional floating-point support (always provided on the PDP-8/A)
is also not part of the PDP-8/X.

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

In order to talk about individual bits,
we speak of the 0x8000_0000 bit (most significant), the 0x4000_0000 bit (slightly less
significant), and so on down to the 0x0000_0001 bit (least significant).  We can use
similar bit masks to talk about multiple bits at the same time.

## Registers

The user-visible registers of a PDP-8/X are very few by modern standards:

 * The AC or accumulator register is 32 bits wide, and is used
   by almost all instructions.
        
 * The MQ or multiplier-quotient register is also 32 bits wide.
   The PDP-8/X does not have multiply or divide operations, so
    for the most part it is just another register, less useful
    than the AC register but occasionally convenient.
     
  * The L or link register is a single bit.  When adding a value to
    the AC, any overflow changes the L register from 0 to 1
    or from 1 to 0.
     
  * The PC or program counter points to the next instruction to be
    executed. Its value is an unsigned integer
    between 0 and however much memory is installed in the system.
     
  The following registers are not visible to programmers and are used
  in this explanation; they may or may not correspond to actual registers:
  
  * The 16-bit IR register holds the instruction currently being executed.
    Note that two instructions are stored in a word.
    IR is always treated as a bit vector.  
    
  * The 4-bit OP register contains the most significant 4 bits of IR.
  
  * The 1-bit S register is 1 if the next instruction is going to be skipped
    (not executed) and 0 otherwise.
       
  * The 32-bit Y register contains the address of the memory location 
    being accessed by the current instruction (if any).
     
   * The 8-bit D register contains the number of an I/O device.
  
   * The 4-bit DOP register contains the number of
     a device-specific I/O instruction.
   
   * The 32-bit H register is set when the machine is built,
     and contains the address of the last existing word in memory.
     
   * The 1-bit B register is used to save a bit while rotating.
           
## Memory
   
Memory is measured in 32-bit words.
It is organized into *pages* that are 2 KW in length.
The page whose addresses are #x0000_0000 to #x0000_03FF inclusive is called
*page zero*.  The page which contains the instruction currently being executed is
called the *current page*.  This use of the term *page* has nothing to do
with virtual memory.
   
The minimum amount of memory for a PDP-8/X is 32 pages or 64 KW.
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

Instructions are fetched, decoded, and executed as follows:

 * Set IR to the least significant halfword of M[PC].
   
 * Set OP to the 4 most significant bits of IR.
   
 * Set the 21 most significant bits of Y
   to the 21 most significant bits of PC,
   and the 11 least significant bits of Y to 0.
   
 * Set PC to PC + 1, ignoring any overflow.
   
 * If PC is greater than H, set PC to 0.
   
 * If S is 0, execute the instruction based on the contents of OP
   and the details in the following sections
   
 * Set S to 0.
   
## Memory-referencing instructions
   
If the value of OP is anything except 0x6, 0x7, 0xE, or 0xF,
IR contains a memory-referencing instruction (MRI), all of which have the same format.
The Y register is set in a common way for all MRIs, and then the value of OP specifies
exactly what to do with Y and the user-visible registers of the PDP-8/X.

The first step is to determine an initial value of Y using the page bit of IR, which is
the 0x0800 bit, and the 11 least significant bits of the IR, which are the 0x07FF bits.
If the page bit is 0, then the 21 most significant bits of Y are set to 0, so that
Y represents an address on page zero.  If the page bit is 1, Y will represent an address
on the current page, and no special action needs to be taken.

Finally, the 11 least significant bits of IR are
copied to the 11 least significant bits of Y.

If the most significant bit of OP is 1, then set Y to M[Y].
This is called *indirect addressing*.  If the bit is 0, it is called
*direct addressing*, and no special action is taken.  Any word in memory can be
accessed indirectly, but only the 2KW of page zero and the 2KW of the current page
can be accessed directly.

Then one of the following six cases is chosen:

 * If OP is 0x0 or 0x8, then set AC to AC bitwise-ANDed with M[Y].
   The assembler mnemonic is AND.
 
 * If OP is 0x1 or 0x9, then set AC to the sum of AC and M[Y].  If
   there is an overflow out of AC as a result, then set L to 1 - L.
   The assembler mnemonic is, for historical reasons, TAD (two's
   complement add).
 
 * If OP is 0x2 or 0xA, then set M[Y] to AC, and then set AC to 0.
   The assembler mnemonic is DCA (deposit and clear AC).
 
 * If OP is 0x3 or 0xB, then set M[Y] to M[Y] + 1; any overflow is ignored.
   If M[Y] is now 0, then set PC  to PC + 1.
   If PC is now greater than H, then set PC to 0.
   The assembler mnemonic is ISZ (increment and skip if zero).
 
 * If OP is 0x4 or 0xC, then set M[Y] to PC, set Q to 0,
   and set PC to Y + 1.
   The assembler mnemonic is JMS (jump to subroutine).
 
 * If is 0x5 or 0xD, then set PC to Y and set Q to 0.
   The assembler mnemonic is JMP.
 
## I/O instructions
 
If OP is 0x6, then set D to the 0x0FF0 bits of IR
and set DOP to the 0x000F bits of IR.
The effect of executing such an instruction depends on
the values of D and DOP, and is documented elsewhere.
If D corresponds to an unknown or unavailable device, or
DOP to an undefined instruction, then no action is taken.
 
## Operate instructions
 
If OP is 0x7 then various operations on AC, L, and/or MQ are
performed depending on the bits of IR,
which are examined in the order given below.
Note that all applicable actions are taken, not just the first one.
 
For convenience, the 0x0010 bit is called the RR (rotate right) bit,
the 0x0004 bit is called the RL (rotate left) bit, 
and the 0x0002 bit is called the RT (rotate twice) bit.

 * If the 0x0800 bit if IR is 1,
   then set the 16 least significant bits of AC
   to the 16 most significant bits of M[PC],
   and the 16 most significant bits of AC
   to the most significant bit of M[PC].
   The assembler mnemonic is IMA (immediate to AC).
   This instruction has no PDP-8 equivalent.

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
   This can be done by rotating left by one bit twice, or more efficiently.
   The assembler mnemonic is RTL (rotate twice left).
   
 * If the RL and RR bits are 0 and the RT bit is 1, then exchange the
   most significant halfword of AC with the 16
   least significant halfword  of AC.
   The assembler mnemonic is HSW (halfword swap); there is no PDP-8 equivalent.
   
 * If the RL, RR, and RT bits are all 1, then set AC to PC.
   The assembler mnemonic is PCA (PC to AC); this is a PDP/8-A
   instruction without its own mnemonic.
 
 * If the RL and RR bits are 1 and the RT bit is 0, then exchange the
   0xF000 and the 0x0F00 bits of AC and then exchange
   0x00F0 and the 0x000F bits of AC.
   The assembler mnemonic is BSW (byte swap); the PDP-8 equivalent
   swaps the two six-bit halfwords and so is more like the HSW instruction.
   
 * If the 0x0080 bit is 1, then set AC to AC bitwise-ORed with MQ.
   The assembler mnemonic is MQA (MQ to AC).
 
 * If the 0x0008 bit is 1, then set MQ to AC.
   The assembler mnemonic is ACM (AC to MQ).

 * If the 0x0080 and the 0x0008 bits are both 1, then exchange MQ and AC.
   The assembler mnemonic is SWP.
    
## Skip instructions
 
If OP is 0xF, then S is set depending on various bits
of IR, AC, and L.
The bits of IR are examined in the order given below.
Note that all applicable actions are taken, not just the first one.

 * If the 0x0200 bit of IR is 1, then set AC to 0.
   The assembler mnemonic is CLA (clear AC).
 
 * If the 0x0100 bit of IR is 1 and the sign (0x80000000) bit of AC is 1,
   (that is, AC is negative), then set S to 1.
   The assembler mnemonic is SMA (skip if minus AC).
  
 * If the 0x0040 bit of IR is 1 and AC is 0, then set S to 1.
    The assembler mnemonic is SZA (skip if zero AC).
 
 * If the 0x0020 bit of IR is 1 and L is 1, then set S to 1.
   The assembler mnemonic is SNL (skip if non-zero L).
 
 * If the 0x0010 bit of IR is 1, then set S to 1 - S.
   The assembler mnemonics for the previous three instructions
   when combined with this bit are SPA (skip if positive-or-zero AC),
   SNA (skip if non-zero AC), and SZL (skip if zero L) respectively.
  
 * If the 0x0002 bit of IR is 1, then halt the PDP-8/X processor.
   When the processor is restarted externally, all registers
   (user-visible and not) and all of memory are unchanged.
   The assembler mnemonic is HLT.
 
## Other instructions

If OP is 0xE, no action is taken. All such instructions are reserved
for possible future use.
