	 / _____)             _              | |    
	( (____  _____ ____ _| |_ _____  ____| |__  
	 \____ \| ___ |    (_   _) ___ |/ ___)  _ \ 
	 _____) ) ____| | | || |_| ____( (___| | | |
	(______/|_____)_|_|_| \__)_____)\____)_| |_|
	  (C)2013 Semtech-Cycleo

LoRa concentrator HAL user manual
============================

1. Introduction
---------------

The LoRa concentrator Hardware Abstraction Layer is a C library that allow you
to use a Semtech concentrator chip through a reduced number of high level C
functions to configure the hardware, send and receive packets.

The Semtech LoRa concentrator is a digital multi-channel multi-standard packet
radio used to send and receive packets wirelessly using LoRa or FSK modulations.

2. Components of the library
----------------------------

The library is composed of 6 modules:

* loragw_hal
* loragw_reg
* loragw_mcu
* loragw_com
* loragw_aux
* loragw_radio

The library also contains basic test programs to demonstrate code use and check
functionality.

### 2.1. loragw_hal ###

This is the main module and contains the high level functions to configure and
use the LoRa concentrator:

* lgw_board_setconf, to set the configuration of the concentrator 
* lgw_rxrf_setconf, to set the configuration of the radio channels
* lgw_rxif_setconf, to set the configuration of the IF+modem channels
* lgw_txgain_setconf, to set the configuration of the concentrator gain table
* lgw_start, to apply the set configuration to the hardware and start it
* lgw_stop, to stop the hardware
* lgw_receive, to fetch packets if any was received
* lgw_send, to send a single packet (non-blocking, see warning in usage section)
* lgw_status, to check when a packet has effectively been sent

For an standard application, include only this module.
The use of this module is detailed on the usage section.

/!\ When sending a packet, there is a 1.5 ms delay for the analog circuitry to
start and be stable (TX_START_DELAY).

In 'timestamp' mode, this is transparent: the modem is started 1.5ms before the
user-set timestamp value is reached, the preamble of the packet start right when
the internal timestamp counter reach target value.

In 'immediate' mode, the packet is emitted as soon as possible: transferring the
packet (and its parameters) from the host to the concentrator takes some time,
then there is the TX_START_DELAY, then the packet is emitted.

In 'triggered' mode (aka PPS/GPS mode), the packet, typically a beacon, is 
emitted 1.5ms after a rising edge of the trigger signal. Because there is no
way to anticipate the triggering event and start the analog circuitry
beforehand, that delay must be taken into account in the protocol.

### 2.2. loragw_mcu ###

This module wraps the HAL functions into commands to be sent to the concentrator
MCU.

The HAL structures are serialized in a byte array and sent over the COM/USB
interface.

* board configuration
* send/receive packets
* ...

### 2.3. loragw_reg ###

This module is used to access to the LoRa concentrator registers by name instead
of by address:

* lgw_connect, to initialise and check the connection with the hardware
* lgw_disconnect, to disconnect the hardware
* lgw_soft_reset, to reset the whole hardware by resetting the register array
* lgw_reg_check, to check all registers vs. their default value and output the
result to a file
* lgw_reg_r, read a named register
* lgw_reg_w, write a named register
* lgw_reg_rb, read a name register in burst
* lgw_reg_wb, write a named register in burst

This module handles pagination, read-only registers protection, multi-byte
registers management, signed registers management, read-modify-write routines
for sub-byte registers and read/write burst fragmentation to respect SPI
maximum burst length constraints.

It make the code much easier to read and to debug.
Moreover, if registers are relocated between different hardware revisions but
keep the same function, the code written using register names can be reused "as
is".

If you need access to all the registers, include this module in your
application.

**/!\ Warning** please be sure to have a good understanding of the LoRa
concentrator inner working before accessing the internal registers directly.

### 2.4. loragw_com ###

This module contains the functions to access the LoRa concentrator register
array through the USB/UART interface:

* lgw_com_r to read one byte
* lgw_com_w to write one byte
* lgw_com_rb to read two bytes or more
* lgw_com_wb to write two bytes or more

Please *do not* include that module directly into your application.

**/!\ Warning** Accessing the LoRa concentrator register array without the
checks and safety provided by the functions in loragw_reg is not recommended.

### 2.5. loragw_aux ###

This module contains a single host-dependant function wait_ms to pause for a
defined amount of milliseconds.

The procedure to start and configure the LoRa concentrator hardware contained in
the loragw_hal module requires to wait for several milliseconds at certain
steps, typically to allow for supply voltages or clocks to stabilize after been
switched on.

An accuracy of 1 ms or less is ideal.
If your system does not allow that level of accuracy, make sure that the actual
delay is *longer* that the time specified when the function is called (ie.
wait_ms(X) **MUST NOT** before X milliseconds under any circumstance).

If the minimum delays are not guaranteed during the configuration and start
procedure, the hardware might not work at nominal performance.
Most likely, it will not work at all.

### 2.6. loragw_radio ###

This module contains functions to handle the configuration of SX125x radios.


3. Software build process
--------------------------

### 3.1. Details of the software ###

The library is written following ANSI C conventions but using C99 explicit
length data type for all data exchanges with hardware and for parameters.

The loragw_aux module contains POSIX dependant functions for millisecond
accuracy pause.
For embedded platforms, the function could be rewritten using hardware timers.

### 3.2. Building options ###

All modules use a fprintf(stderr,...) function to display debug diagnostic
messages if the DEBUG_xxx is set to 1 in library.cfg

### 3.3. Building procedures ###

For cross-compilation set the ARCH and CROSS_COMPILE variables in the Makefile,
or in your shell environment, with the correct toolchain name and path.
ex:
export PATH=/home/foo/rpi-toolchain/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin:$PATH
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-

The Makefile in the libloragw directory will parse the library.cfg file and 
generate a config.h C header file containing #define options.
Those options enables and disables sections of code in the loragw_xxx.h files 
and the *.c source files.

The library.cfg is also used directly to select the proper set of dynamic 
libraries to be linked with.

### 3.4. Export ###

Once build, to use that library on another system, you need to export the
following files :

* libloragw/library.cfg  -> root configuration file
* libloragw/libloragw.a  -> static library, to be linked with a program
* libloragw/readme.md  -> required for license compliance
* libloragw/inc/config.h  -> C configuration flags, derived from library.cfg
* libloragw/inc/loragw_*.h  -> take only the ones you need (eg. _hal and _gps)

After statically linking the library to your application, only the license 
is required to be kept or copied inside your program documentation.

4. Hardware dependencies
------------------------

### 4.1. Hardware revision ###

The loragw_reg and loragw_hal are written for a specific version on the Semtech
hardware (IP and/or silicon revision).

This code has been written for:

* Semtech SX1301 chip
* Semtech SX1257 or SX1255 I/Q transceivers

The library will not work if there is a mismatch between the hardware version 
and the library version. You can use the test program test_loragw_reg to check 
if the hardware registers match their software declaration.

### 4.2. USB/UART communication ###

loragw_com contains 4 functions (read, write, burst read, burst write) that are
platform-dependant.
The functions must be rewritten depending on the communication bridge you use:

* USB/UART over linux tty port (provided)

You can use the test program test_loragw_com to check that the USB communication
is working.


5. Usage
--------

### 5.1. Setting the software environment ###

For a typical application you need to:

* include loragw_hal.h in your program source
* link to the libloragw.a static library during compilation
* link to the librt library due to loragw_aux dependencies (timing functions)

For an application that will also access the concentrator configuration 
registers directly (eg. for advanced configuration) you also need to:

* include loragw_reg.h in your program source

### 5.2. Using the software API ###

To use the HAL in your application, you must follow some basic rules:

* configure the radios path and IF+modem path before starting the radio
* the configuration is only transferred to hardware when you call the *start*
  function
* you cannot receive packets until one (or +) radio is enabled AND one (or +)
  IF+modem part is enabled AND the concentrator is started
* you cannot send packets until one (or +) radio is enabled AND the concentrator
  is started
* you must stop the concentrator before changing the configuration

A typical application flow for using the HAL is the following:

	<configure the radios and IF+modems>
	<start the LoRa concentrator>
	loop {
		<fetch packets that were received by the concentrator>
		<process, store and/or forward received packets>
		<send packets through the concentrator>
	}
	<stop the concentrator>

**/!\ Warning** The lgw_send function is non-blocking and returns while the
LoRa concentrator is still sending the packet, or even before the packet has
started to be transmitted if the packet is triggered on a future event.
While a packet is emitted, no packet can be received (limitation intrinsic to
most radio frequency systems).

Your application *must* take into account the time it takes to send a packet or 
check the status (using lgw_status) before attempting to send another packet.

Trying to send a packet while the previous packet has not finished being send
will result in the previous packet not being sent or being sent only partially
(resulting in a CRC error in the receiver).

### 5.3. Debugging mode ###

To debug your application, it might help to compile the loragw_hal function
with the debug messages activated (set DEBUG_HAL=1 in library.cfg).
It then send a lot of details, including detailed error messages to *stderr*.

6. License
-----------

Copyright (c) 2013, SEMTECH S.A.
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright
  notice, this list of conditions and the following disclaimer.
* Redistributions in binary form must reproduce the above copyright
  notice, this list of conditions and the following disclaimer in the
  documentation and/or other materials provided with the distribution.
* Neither the name of the Semtech corporation nor the
  names of its contributors may be used to endorse or promote products
  derived from this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL SEMTECH S.A. BE LIABLE FOR ANY
DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
(INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND
ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

*EOF*
