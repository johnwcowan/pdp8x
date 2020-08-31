# PALX assembler for the PDP-8/X
Like most assemblers, PALX is line oriented.
However, a semicolon is equivalent to a line break,
except inside a comment.

## Line layouts
```
[label,] expr* [/comment]
[label,] mri [I] [Z] addr [/comment]
[label,] pseudo-op [/comment]
```

The memory-referencing instructions are TBD.

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
(whitespace) - bitwise or
# - low half on the left, high half on the right
```

## Pseudo-ops
```
*addr - set assembly address (absolute or relocatable)
name = expr - define name
base addr - set the assembler's idea of the base page
block n [k] - assemble a block of n words filled with k (default 0)
common [name] - start assembling a new common section
commz [name] - start assembling a new common section in page zero
cpage n - ensure the next n words are on the current page
eap - enter auto-paging (CPU code can cross pages)
extern name - name is external (may be defined or undefined)
fsect [name] - start assembling a new relocatable section with FPP code
index addr - set the assembler's idea of the index register block
lap - leave auto-paging (CPU code cannot cross pages)
page [n] - start assembling at the beginning of page n, or next page
reloc addr - assemble in current location as if at addr
sect [name] - start assembling a new relocatable section with CPU code
skipdef name=addr - define new skip instruction
text /foo/ - assemble UTF-8 into words with NUL termination (little-endian)
```

## Notes
INC is equivalent to ISZ, but the programmer guarantees it won't skip.
(FPP equivalents needed?)

