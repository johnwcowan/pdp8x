# PDP-8/X devices

D is the 8-bit register specifying which device to operate.  
DOP is the 4-bit register specifying what operation to execute.

## Console keyboard (D = 0x03)

This corresponds to the KL8-E console keyboard controller.

## Console text display (D = 0x04)

This corresponds to the KL8-E console text display or printer controller.

## Binary input from source (D = 0x01)

This device corresponds to the PR8-E high-speed paper tape reader controller,
and is connected to a source of bytes.
It has a single 8-bit register RB that holds the last byte read from the source.
There is also a 1-bit register RF indicating that the last operation is complete.

DOP = 1: Skip on reader flag (RSF).
If RF = 1, then S is set to 1 so that the
next instruction will be skipped.

DOP = 2: Read reader buffer (RRB).
Set AC to AC bitwise-ORed with RB
and set RF to 1.

DOP = 4: Fetch character into RB (RFC).
Set RF to 0.
Start to read one byte from the source into RB,
which can be done asynchronously.
When the byte is read, set RF to 1.

DOP = 6: Read buffer and fetch character (RRB RFC).
Set AC to AC bitwise-ORed with RB.
Set RF to 0.
Start to read one byte from the source,
which can be done asynchronously.
When the byte is read, set RF to 1.

## Binary output to sink (D = 0x02)

This device corresponds to the PC8-E high-speed paper tape punch controller,
and is connected to a sink for bytes.
It has a single 8-bit register PB that holds the last byte to be written to the sink.
There is also a 1-bit register PF indicating that the last operation is complete.

DOP = 1: Skip on punch flag  (PSF).
If PF = 1, then S is set to 1 so that the
next instruction will be skipped.

DOP = 2: Clear punch flag (PCF).
Set PF to 0.

DOP = 4: Load punch buffer and punch character (PPC).
Set PB to the 8 least significant bytes of AC.
Start to write one byte from PB into the sink,
which can be done asynchronously.
When the byte is written, set PF to 1.

DOP = 6: Load punch buffer sequence (PLS).
Set PB to the 8 least significant bytes of AC.
Start to write one byte from PB into the sink,
which can be done asynchronously.
When the byte is written, set PF to 1.

## Graphical display (D = 0x05)

This corresponds loosely to the VC8-E oscilloscope controller,
but behaves more like a turtle-graphics window.

## External block storage

This corresponds to the RK8-E removable disk cartridge controller.
