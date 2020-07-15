# PDP-8/X devices

D is the 8-bit register specifying which device to operate.  
DOP is the 4-bit register specifying what operation to execute.

## Console keyboard (D = 0x03)

This corresponds to the KL8-E console text display or printer controller.
and is connected to a source of characters.
It has a single register KB that holds the last character read from the source.
The width of this register depends on where the input is coming from.
If the source provides Unicode characters, it will be 21 bits wide,
but if the source provides only ASCII characters, it will be 8 bits wide
and the most significant bit will always be 1.
There is also a 1-bit register KF indicating that the last operation is complete.

DOP = 0: Clear keyboard flag (KCF).
Set KF to 0.

DOP = 1: Skip on keyboard flag (KSF).
If KF = 1, then set S to 1 so that the
next instruction will be skipped.

DOP = 2: Clear keyboard combined (KCC).
Set AC and KF to 0.
Start to read one character from the source into RB,
which can be done asynchronously.
When the character is read, set RF to 1.

DOP = 4: Read keyboard buffer static (KRS).
Set AC to AC bitwise-ORed with KB.

DOP = 6: Read keyboard buffer dynamic (KRB).
Set AC to KB, clearing the most significant bits of AC.
Set KF to 0.
Start to read one character from the source,
which can be done asynchronously.
When the character is read, set RF to 1.

## Console text display (D = 0x04)

This corresponds to the KL8-E console text display or printer controller.
and is connected to a sink for bytes.
It has a single register TB that holds the next character to be written to the source.
The width of this register depends on where the output is going to.
If the sink accepts Unicode characters, it will be 21 bits wide,
but if the sink accepts only ASCII characters, it will be 8 bits wide,
and the most significant bit will be ignored.
There is also a 1-bit register TF indicating that the last operation is complete.

DOP = 0: Set text flag (TFL).
Set TF to 1.

DOP = 1: Skip on text flag  (TSF).
If TF = 1, set S to 1 so that the
next instruction will be skipped.

DOP = 2: Clear text flag (TCF).
Set TF to 0.

DOP = 4: Load text buffer and output character (TCP).
Set TB to the least significant bits of AC.
Start to write one character from TB into the sink,
which can be done asynchronously.
When the character is written, set TF to 1.

DOP = 5: Skip on text or keyboard flag (TSK).
If TF = 1 or KF = 1, set S to 1 so that the
next instruction will be skipped.

DOP = 6: Load teleprinter sequence (TLS).
Set TF to 0.
Set TB to the  least significant bits of AC.
Start to write one character from PB into the sink,
which can be done asynchronously.
When the character is written, set PF to 1.

## Binary input from source (D = 0x01)

This device corresponds to the PR8-E high-speed paper tape reader controller,
and is connected to a source of bytes.
It has a single 8-bit register RB that holds the last byte read from the source.
There is also a 1-bit register RF indicating that the last operation is complete.

DOP = 1: Skip on reader flag (RSF).
If RF = 1, then set S to 1 so that the
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
It has a single 8-bit register PB that holds the next byte to be written to the sink.
There is also a 1-bit register PF indicating that the last operation is complete.

DOP = 1: Skip on punch flag  (PSF).
If PF = 1, set S to 1 so that the
next instruction will be skipped.

DOP = 2: Clear punch flag (PCF).
Set PF to 0.

DOP = 4: Load punch buffer and punch character (PPC).
Set PB to the 8 least significant bits of AC.
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
