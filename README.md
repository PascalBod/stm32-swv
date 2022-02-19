# Using SWV with an STM32 microcontroller

## Context

Recently, I was adding a new functionality to an existing application written for an STM32L476RG. This application uses version 2.1.0 of ST's LoRaWAN stack, and the trace utility delivered with the stack. With the trace utility, it sends messages on UART2, and accepts commands on this UART as well.

After some time, while testing the application, I discovered a problem: sending a specific command to the microcontroller was making it ignore additional commands, right after the application had replied with an acknowledgement message. The debugger allowed me to find a reason for this: the reception, which was interrupt-driven, was inhibited some time around the transmission of the acknowledgement message. I was able to observe that the `RXNEIE` bit of USART2's `CR1` register was reset to `0`, instead of being kept at `1`.

Now, the question I had to answer: when was the bit reset? Running the code step by step or with some breakpoints was not the right way to find the answer, as it rapidly appeared that the problem was due to relative timing of some events: it was not occurring if the reception of the command or the transmission of the acknowledgement message were delayed by a breakpoint. Of course, adding more trace messages had the same impact.

I had heard about SWV before, but had never got the opportunity to use it. It seemed that it was time to play with it.

## SWD and SWV

SWD (Serial Wire Debug) is the Arm's alternative to JTAG, to debug embedded code. Only two wires are used, instead of (a minimum of) four wires for JTAG: SWDCLK and SWDIO. A third wire, SWO, allows the microcontroller to output additional debugging data. SWO is available on Cortex-M3 and Cortex-M4 cores, but not on Cortex-M0/M0+.

SWV (Serial Wire Viewer) is an application running on the debugging host, wich provides the following features, thanks to SWO:
* Program counter sampling
* Event counters that show CPU cycle statistics
* Exception and Interrupt execution with timing statistics
* Trace data - data reads and writes used for timing analysis
* Trace information used for simple printf-style debugging

A probe is required, to connect to the microcontroller board. On my side, I develop the application for the NUCLEO-L476RG board, and the probe (ST-LINK/V2) is on the board.

For software development and debugging, I use STM32CubeIDE.

## How to setup the debugging environment






