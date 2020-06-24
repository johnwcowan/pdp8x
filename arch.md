# The PDP-8/X architecture

The PDP-8/X is a rethink of the DEC PDP-8, a 12-bit minicomputer.
Because the PDP-8 is word-oriented, it almost doesn't matter
how big the words are, so we extend them from 12 bits to 32 bits.
However, the instructions are only 16 bits long, so there are two
of them in a single word.

Good resources for the final PDP-8 models, the PDP-8/E and the
PDP-8/A (the letters are meaningless and not even alphabetical), are the
*[PDP-8/E and PDP/8-M Small Computer Handbook 1972](https://www.grc.com/pdp-8/docs/PDP-8_Small_Computer_Handbook_1972.pdf)*
and the *[PDP/8-A Miniprocessor Users Manual 1976](http://www.vandermark.ch/pdp8/uploads/PDP8/PDP8.Manuals/EK-8A002-MM-002.pdf)*.
This document is much less comprehensive, and will attempt to explain
as clearly as possible what a PDP-8/X CPU actually appears to do.
(Of course the implementation may be quite different, provided the
registers and memory locations are the same.)

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
between -2,147,483,648 and 2,147,483,647 inclusive.
The representation of the integer is twos-complement,
like most machines today. That is, a word is negated
by bitwise-NOTting it and then adding 1 to the result.

We write numeric values either as signed decimal numbers
or as hexadecimal values starting with 0x.
This is different from PDP-8 architecture descriptions,
which use octal numbers, grouping the 12 bits as 4 groups of 3.

## Registers

The user-visible registers of a PDP-8/X are very few by modern standards:

 * The AC or accumulator register is 32 bits wide, and is used
   by almost all instructions.
        
 * The MQ or multiplier-quotient register is also 32 bits wide.
   The PDP-8/X does not have multiply or divide operations, so
    for the most part it is just another register, much less useful
    than the AC register.
     
  * The L or link register is a single bit.  When adding a value to
    the AC, any overflow changes the L register from 0 to 1
    or from 1 to 0.
     
  * The PC or program counter points to the next instruction to be
    executed. Its value is an unsigned integer
    between 0 and however much memory is installed in the system.
     
  The following registers are not visible to programmers and are used
  in this explanation; they may or may not correspond to actual registers:
  
  * The 16-bit IR register holds the instruction currently being executed.
    It is always treated as a bit vector.  In order to talk about individual bits,
    we speak of the 0x8000 bit (most significant), the 0x4000 bit (slightly less
    significant), and so on down to the 0x0001 bit (least significant).  We can use
    similar bit masks to talk about multiple bits at the same time.
    
  * The 4-bit OP register contains the most significant 4 bits of IR.
  
  * The 1-bit S register is 1 if the next instruction is going to be skipped
    (not executed) and 0 otherwise.
    
  * The 32-bit Y register contains the address of the memory location being accessed by the
     current instruction (if any).
     The notation M[Y] specifies the memory location whose address is Y.
     
   * The 8-bit D register contains the number of an I/O device.
  
   * The 4-bit DOP registercontains the number of a device-specific I/O instruction.
   
   * The 32-bit H register is set when the machine is built, and contains the address of the last
     existing word in memory.
      
## Memory
   
Memory is measured in kibiwords (KW).  1 word is 4 bytes and 1 KW is 1024 words.
Memory is organized into *pages* that are 2 KW in length, addressed from 0 upwards.
The page whose addresses are #x00000000 to #x000007FF inclusive is called
*page zero*.  The page which contains the instruction currently being executed is
called the *current page*.
   
The minimum amount of memory for a PDP-8/X is 16 pages or 32 KW.
The maximum amount is 2,097,152 pages or 4 GW; in a fully equipped
machine the address of the last word is 0xFFFFFFFF.  A read from
a non-existent page returns 0; a write to a non-existent page does nothing.

(To determine if a page exists at run time, read a word from the page.
If the value is not 0, the page exists.  Otherwise, write a non-zero value
to the same word and read it a second time.  If the value is 0, the page
does not exist.  Otherwise the page exists; write 0 to the word.)

Because there are two instructions per word and all addressing is by words,
it is sometimes necessary to have a no-operation instruction (0x7000)
as the second instruction of a word.  The PDP-8/X assembler can insert
such instructions before a label and after a JMS instruction (see below)
when needed, as it is only possible to
jump to the first instruction of a word.
   
**It is not yet specified whether the first instruction is stored
in the more significant (0xFFFF0000) or the less significant (0x0000FFFF)
bits of the word.**

## Instruction decoding

Each cycle of the PDP-8/X begins by setting IR to M[PC] and setting OP from the four
most significant bits (0xF000) of IR.
The 21 most significant bits of PC (0xFFFFF800) are copied to Y, and the 11 least significant
bits of Y are set to 0.
Then PC is set to PC + 1, but if PC is equal to H, then set PC to 0 instead.
      
## Memory-referencing instructions
   
Every instruction except the ones whose OP register contains 0x6, 0x7, 0xE, or 0xF
is a memory-referencing instruction (MRI), all of which have the same format.
The Y register is set in a common way for all MRIs, and then the value of OP specifies
exactly what to do with Y and the user-visible registers of the PDP-8/X.

The first step is to determine an initial value of Y using the page bit of the IR, which is
the 0x0800 bit, and the 11 least significant bits of the IR, which are the 0x07FF bits.
If the page bit is 0, then the 21 most significant bits of Y are set to 0, so that
Y represents an address on page zero.  If the page bit is 1, Y will represent an address
on the current page, and the 21 most significant bits of Y are set to
the 21 most significant bits of PC - 1 (but if PC is 0, it is set to H).
Finally, the 11 least significant bits of IR are copied to the least significant bits of Y.

If the most significant bit of OP (or of IR) is set to 1, then Y is set to M[Y].
Then one of the following six cases is chosen:

 * If OP is 0x0 or 0x8, then AC is set to AC bitwise-ANDed with M[Y].
   The assembler mnemonic is AND.
 
 * If OP is 0x1 or 0x9, then AC is set to the sum of AC and M[Y].  If
   there is an overflow out of AC as a result, then L is set to 1 - L.
   The assembler mnemonic is, for historical reasons, TAD (two's
   complement add).
 
 * If OP is 0x2 or 0xA, then M[Y] is set to AC, and AC is set to zero.
   The assembler mnemonic is DCA (deposit and clear AC).
 
 * If OP is 0x3 or 0xB, then M[Y] is set to M[Y] + 1; any overflow is ignored.
   If M[Y] is now 0, then PC is set to PC + 1,
   but if PC is, then its new value is 0x00000000.
   The assembler mnemonic is ISZ (increment and skip if zero).
 
 * If OP is 0x4 or 0xC, then M[Y] is set to PC and PC is set to Y + 1.
   The assembler mnemonic is JMS (jump to subroutine).
 
 * If is 0x5 or 0xD, then PC is set to Y.
   The assembler mnemonic is JMP.
 
# I/O instructions
 
If OP is 0x6, then D is set to the 0x0FF0 bits of IR
and DOP is set to the 0x000F bits of IR.
The effect of executing such an instruction depends on
the values of D and DOP, and is documented elsewhere.
If D corresponds to an unknown or unavailable device, or
DOP to an undefined instruction, then no action is taken.
 
# Operate instructions
 
If OP is 0x7 then various operations on A, L, and/or MQ are
performed depending on the bits of IR, which are examined
in the order below.
 
For convenience, the 0x0010 bit is called the RR (rotate right) bit,
the 0x0004 bit is called the RL (rotate left) bit, 
and the 0x0002 bit is called the RT (rotate twice) bit.

 * If the 0x0200 bit of IR is 1, then A is set to 0.
   The assembler mnemonic is CLA (clear AC).
  
 * If the 0x0100 bit of IR is 1, then L is set to 0
   The assembler mnemonic is CLL (clear link).

 * If the 0x0040 bit of IR is 1, then A is set to the bitwise-NOT of A,
   or equivalently to (-A) - 1.
   The assembler mnemonic is CMA (complement AC).
   
 * If the 0x0020 bit of IR is 1, then L is set to 1 - L.
   The assembler mnemonic is CLL (clear link).
 
 * If the 0x0001 bit of IR is 1, then A is set to A + 1;
   overflow is ignored.
   The assembler mnemonic is IAC (increment AC).
 
 * If the RR bit of IR is 1 and the RL and RT bits of IR are 0,
   then A and L are jointly rotated right by one bit.  That is, L is saved
   and set to 0, A is shifted left by one bit, and the saved L
   is added to A.
   The assembler mnemonic is RAR (rotate AC right).
   
 * If the RR and RT bits of IR are 1 and the RL bit of IR is 0,
   then A and L are jointly rotated right by two bits.  This can be
   done by rotating right by one bit twice, or more efficiently.
   The assembler mnemonic is RTR (rotate twice right).
   
 * If the RL bit of IR is 1 and the RR and RT bits of IR are 0,
   then A and L are jointly rotated left by one bit.  That is, L is saved
   and set to 0, A is shifted left by one bit, and the saved L
   is added to A.
   The assembler mnemonic is RAL (rotate AC left).
   
 * If the RL and RT bits of IR are 1 and then RR bit of IR is 0,
   then A and L are jointly rotated left by two bits.  This can be
   done by rotating left by one bit twice, or more efficiently.
   The assembler mnemonic is RTL (rotate twice left).
   
 * If the RL and RR bits are 0 and the RT bit is 1, then the
   16 most significant (0xFFFF0000) bits of A are exchanged with the 16
   least significant (0X0000FFFF) bits of A.
   The assembler mnemonic is HSW (halfword swap); there is no PDP-8 equivalent.
   
 * If the RL, RR, and RT bits are all 1, then A is set to PC.
   The assembler mnemonic is PCA (PC to AC); this is a PDP/8-A
   instruction without its own mnemonic.
 
 * If the RL and RR bits are 1 and the RT bit is 0, then the
   0xF000 and the 0x0F00 bits of A are exchanged, and the
   0x00F0 and the 0x000F bits of A are exchanged.
   The assembler mnemonic is BSW (byte swap); the PDP-8 equivalent
   swaps the two six-bit halfwords and so is more like the HSW instruction.
   
 * If the 0x0080 bit is 1, then AC is set to AC bitwise-ORed with MQ.
   The assembler mnemonic is MQA (MQ to AC).
 
 * If the 0x0008 bit is 1, then MQ is set to AC.
   The assembler mnemonic is ACM (AC to MQ).

 * If the 0x0080 and the 0x0008 bits are both 1, then MQ and AC are exchanged.
   The assembler mnemonic is SWP.
    
# Skip instructions
 
If OP is 0xF, then PC is increased by 1 depending on various bits.
The S register is set to 0 at the start of decoding.

 * If the 0x0200 bit of IR is 1, then AC is set to 0.
   The assembler mnemonic is CLA (clear AC).
 
 * If the 0x0100 bit of IR is 1 and the sign (0x80000000) bit of AC is 1,
   (that is, AC is negative), then S is set to 1.
   The assembler mnemonic is SMA (skip if minus AC).
  
 * If the 0x0040 bit of IR is 1 and AC is 0, then S is set to 1.
    The assembler mnemonic is SZA (skip if zero AC).
 
 * If the 0x0020 bit of IR is 1 and L is 1, then S is set to 1.
   The assembler mnemonic is SNL (skip if non-zero L).
 
 * If the 0x0010 bit of IR is 1, then S is set to 1 - S.
   The assembler mnemonics for the previous three instructions
   when combined with this bit are SPA (skip if positive-or-zero AC),
   SNA (skip if non-zero AC), and SZL (skip if zero L) respectively.
 
 * If S is set to 1, then set PC to PC + 1, but if PC is equal to H,
   then set PC to 0.
 
 * if the 0x0002 bit of IR is 1, then halt the PDP-8/X processor.
   The assembler mnemonic is HLT.
 
# Other instructions

If OP is 0xE, then the instructions are reserved and no action is taken.