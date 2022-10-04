# PALX assembler for the PDP-8/X
Like most assemblers, PALX is line oriented.
However, a semicolon is equivalent to a line break,
except inside a comment.

## Line layouts
```
[label,] expr* [/comment]
[label,] nonmri [/comment]
[label,] mri [I] [Z] addr [/comment]
[label,] pseudo-op [/comment]
```

# Expression primitives
```
name - value of name
number without decimal point - hex integer
number with trailing decimal point - decimal integer
number with non-trailing decimal or exponent - 32-bit IEEE float
"..." - 1 to 4 character constant (packed little-endian)
(expr) - address of newly allocated word on this page whose value is expr
[expr] - address of newly allocated word on pagef zero whose value is expr
. - current address
```

# Expression operators
```
+ - integer addition
- integer subtraction/negation
& - bitwwise and
| - bitwise or
# - low half on the left, high half on the right
```

## Mixing absolute and relocatable values

Constants and non-label names are absolute, labels are relocatable.

```
absolute <op> absolute = absolute
relocatable + absolute = relocatable
relocatable - absolute = relocatable
relocatable - relocatable = absolute
```

Everything else is an error.

## Pseudo-ops
```
*addr - set assembly address (absolute or relocatable)
name = expr - define name
asect [name] - start assembling a new absolute section
base addr - set the assembler's idea of the FPP base page
block n [k] - assemble a block of n words filled with k (default 0)
common [name] - start assembling a new common section
commz [name] - start assembling a new common section in page zero
cpage n - ensure the next n words are on the current page
eap - enter auto-paging (CPU code can cross pages)
extern name - name is external (may be defined or undefined)
index addr - set the assembler's idea of the FPP index register block
lap - leave auto-paging (CPU code cannot cross pages)
page [n] - start assembling at the beginning of page n, or next page
reloc addr - assemble in current location as if at addr
sect [name] - start assembling a new relocatable section with CPU code
skipdef name=addr - define new skip instruction
text /foo/ - assemble UTF-8 into words with NUL termination (little-endian)
word - assemble 0x0000 if high half, nothing if low half
       (implied after jump instructions or before datums or labeled instructions)
```

## Notes
INC is equivalent to ISZ, but the programmer guarantees it won't skip.
(FPP equivalents needed?)

# Page zero subroutine for off-page indirection in EAP mode

```
          *07FA
saveac,   0
indirect, 0
offpage,  0
          dca saveac     / save AC
          tad i offpage  / fetch word after call to offpage
          isz offpage    / on return, skip word just fetched
          dca indirect   / stash the address to indirect through
          tad saveac     / restore AC
          jmp i offpage  / return from subroutine; last instruction on page zero
```

The calling sequence is:
```
          jms i offpage
          address
          <mri> i indirect
```

where <mri> is the operation we want to do.

# Instruction sequences at end of code area in EAP mode

```
          / last instruction, non-skip
          jmp last
          / assembler-inserted pointers and values
last,     nop            / last instruction on page
```
                    

```
          / last instruction, skip
          jmp last-1
          jmp last
          / assembler-inserted pointers and values
          skp
last,     skp            / last instruction on page
```

