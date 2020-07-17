# PDP-8/X devices

D is the 8-bit register specifying which device to operate.  
DOP is the 4-bit register specifying what operation to execute.  
For example, an 0x6123 instruction would set D to 0x12 and DOP to 3.

## Console keyboard (D = 0x03)

This corresponds to the KL8-E console text display or printer controller.
and is connected to a source of characters.
It has a single register KB
that holds the last character read from the source.
The width of this register depends on where the input is coming from.
If the source provides Unicode characters, it will be 21 bits wide,
but if the source provides only ASCII characters, it will be 8 bits wide
and the most significant bit will always be 1.
There is also a 1-bit register KF
indicating that the last operation is complete.

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
It has a single register TB
that holds the next character to be written to the source.
The width of this register depends on where the output is going to.
If the sink accepts Unicode characters, it will be 21 bits wide,
but if the sink accepts only ASCII characters, it will be 8 bits wide,
and the most significant bit will be ignored.
There is also a 1-bit register TF
indicating that the last operation is complete.

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
It has a single 8-bit register RB
that holds the last byte read from the source.
There is also a 1-bit register RF
indicating that the last operation is complete.

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
It has a single 8-bit register PB
that holds the next byte to be written to the sink.
There is also a 1-bit register PF
indicating that the last operation is complete.

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

This device corresponds loosely to the VC8-E oscilloscope controller,
but behaves more like a turtle-graphics window.
It is connected to a graphics screen.
It has two 32-bit registers X and Y, which are both 0
to address the pixel in the center of the screen.
There is also a 1-bit register GF
indicating that the last operation is complete.

Depending on the physical pixel count of the screen,
the least significant bits of X and Y may be ignored.
Also note that the device's pixels are square,
which means they do not necessarily correspond
to physical pixels on the screen.

The device also maintains a notion of the current pixel,
which logically corresponds to the location of an invisible turtle.

DOP = 0: Clear all (DILC).

Set X and Y and GF to 0 and clear the screen.

DOP = 1: Clear done flag (DICD).

Set GF to 0.

DOP = 2: Skip on done flag (DISD).

If GF = 1, set S to 1 so that the
next instruction will be skipped.

DOP = 3: Load X register (DILX).

Set X to AC.

DOP = 4: Load Y register (DILY).

Set Y to AC.

DOP = 5: Draw line (DIDR).

Set GF to 0, and draw a line from the current pixel
to the pixel addressed by X and Y,
which can be done asynchronously.
When the line is drawn and the current pixel
has been changed to be the pixel at the end of the line,
set GF to 1.

DOP = 6: Move (DIMV).

Set GF to 0, and change the current pixel
to be the pixel addressed by X and Y,
which can be done asynchronously.
When the change is complete, set GF to 1.

DOP = 7: Read done flag (DIRE):

Set AC to 0.
Then set the most significant (sign) bit of AC to GF.


## External block storage (D = 0x70)

This corresponds to the RK8-E removable disk cartridge controller,
and is connected to 1-4 randomly accessible devices
(which may be emulated by files on a physical device).
The devices may be set read-only or read/write externally
and may be set read-only by the controller.
Each device is partitioned into tracks,
and each track into 32 sectors of 2 KW (one page) each.
The minimum number of tracks is 200,
and the maximum number of tracks is 2^27^ - 1.

The controller has four registers:
 * DC is the 16-bit command register, whose various bits
   instruct the controller what to do.
 * DS is the 16-bit status register, whose various bits
   report on the state of the controller.
 * DD is the 32-bit disk address register, which contains
   the track (the most significant 27 bits) and the sector
   (the least significant 5 bits) to be operated on
   according to DC.
 * DA is the 32-bit memory address register, which contains
   the memory address to which data is read or from which
   it is written.

The 0x7000 bits of DC contain the command to be executed
by the next DLAG instruction.  The command 0 means that data
is to be read, 2 that the disk is to be set read-only,
3 that the specified track is to be seeked to
but no other action taken,
and 4 that data is to be written.

The 0x0200 bit of DC indicates whether a completed seek
sets the 0x8000 (sign) bit of DS to 0 or whether it leaves
the bit unchanged.

The 0x0060 bits of DC specify the device that is to be operated on
by the next DLAG instruction, and whose status is retrieved
by the next DRST instruction.

The 0x8000 (sign) bit of DS is 1
if the device has not yet completed the current operation,
and is 0 if it has.  Whether seeking counts as an operation
depends on the 0x0200 bit of DC.

The 0x0FFF bits of DS indicate various errors; if they are
all 0, there is no error.
Specifically, the 0x0200 bit is 1 if the selected device
does not exist,
the 0x0020 bit is 1 if the selected device has been set read-only,
the 0x0001 bits is 1 if the specified track does not exist
on the selected device.
The meaning of the other bits is device-dependent.

DOP = 1: Disk skip on flags (DSKP).

If the 0x8000 (sign) bit of DS is 0 or the
0x0FFF bits of DS are not 0, then set S to S + 1

DOP = 2: Disk clear (DCLD).

If the least significant bit of AC = 0,
set AC and DS to 0.
If the least significant bit of AC = 1,
set AC, DC, DS, DD, and DA to 0.
Any action in progress is stopped.

DOP = 3: Load address and go (DLAG).

Set DD to AC and
set the 0x8000 bit of DS to 1.
specified in DC,
which can be done asynchronously.
When the action is complete,
set the 0x8000 bit of DS to 0

DOP = 4: Load current address (DLCA).

Set DA to AC.

DOP = 5: Read DS (DRST).

Set AC to the 16 least significant bits of DS.

DOP = 6: Write DC (DLDC).

Set DC to the 16 least significant bits of AC.
