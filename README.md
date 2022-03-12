# Using SWV with an STM32 microcontroller

## Overview

SWV (Serial Wire Viewer) is a trace feature that can be used with many ARM Cortex-M3, M4 and higher processors (but not with Cortex-M0 and Cortex-M0+). SWV trace information can be sent out using a dedicated pin, SWO (Serial Wire
Output) pin, to the debugging host. The following features are provided:
* Program counter sampling
* Event counters that show CPU cycle statistics
* Exception and Interrupt execution with timing statistics
* Trace data - data reads and writes used for timing analysis
* Trace information used for simple printf-style debugging

SWV can be activated when debugging with SWD (Serial Wire Debug), the Arm's alternative to JTAG.

This article presents an overview of the configuration and use of SWV, for an STM32 Nucleo board, the [NUCLEO-L476RG](https://www.st.com/en/evaluation-tools/nucleo-l476rg.html). This board integrates an ST-LINK programmer/debugger, so there is no need for a separate JTAG/SWD probe. [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html) version 1.8.0 is used as development environment.

## Prerequisites

* A NUCLEO-L476RG board (see above)
* STM32CubeIDE version 1.8.0 (see above)
* some experience of STM32CubeIDE: creating a new project, building it, creating a debug configuration, debugging

## Creating an SWV-enabled application

### Hardware

For STM32 microcontrollers, SWD is mapped to PA13 (SWDIO), PA14 (SWCLK) and PB3 (SWO). 

When creating a new project, CubeMX sets PA13 and PA14 as active (green color in CubeMX view below), while PB3 is reserved but inactive (orange color). 

A good thing is to declare PB3 as active, in order to protect it from being selected for another use, so that SWO can be used.

![](images/SWVPins01.png)

This is done by selecting **Trace Asynchronous Sw** for **Debug** in **System Core > SYS**, in **Pinout & Configuration** tab, for `ioc` file generated by CubeMX. PB3 color switches to green.

While in CubeMX, check the value of HCLK, in the **Clock Configuration** tab:

![](images/HCLK01.png)

This value will have to be used when configuring SWV.

Save the `ioc` file, generate code and build the project.

### Software

#### Displaying exceptions with SWV

Create an empty STM32 project, and configure it as presented in the above section.

Create a debug configuration for the project. In the **Debugger** tab, tick **Enable** in the **Serial Wire Viewer** pane. Then set **Core Clock** to the value of HCLK. For me, Core Clock was set to 16.0 by default. I set it to the value of HCLK: 80.

![](images/SWVConfig01.png)

Start the debug session.

The execution stops at the first instruction after `main()`. Select **Window > Show View > SWV > SWV Exception Trace Log**. Click on the **SWV Exception Trace Log** view that has been added to the tabbed notebook containing the **Console** view.

Then click on the settings icon (wrench + screwdriver), on the right-hand side.

![](images/SWVTools01.png)

In the newly displayed view, tick **EXETRC: Trace Exceptions**. Click on the **OK** button.

![](images/SWVEXETRC01.png)

Last step: click on the red button, at the right of the settings icon. This instructs SWV to start tracing as soon as the program execution is resumed.

Resume the execution.

The SWV Exception Trace Log view displays the exceptions when they happen. In our example, the only active code is the SysTick handler, called every time the SysTick interruption is triggered. Consequently, the **Data** tab of the SWV Exception Trace Log view only displays periodic calls to the SysTick handler:

![](images/exceptionLogData01.png)

The **Statistics** tab makes clear that SysTick handler is the only active code:

![](images/exceptionLogStatistics01.png)

#### Displaying the value of a variable with SWV

SWV allows to display the value of a variable, almost in real time, without disturbing the execution context of the application.

To test this function, we declare a few variables at the main level of `main.c`:

```C
/* USER CODE BEGIN PV */
static const uint32_t PERIOD = 1000;
static uint32_t prev_ticks = 0;
static uint32_t new_ticks;
/* USER CODE END PV */
```

Then, we modify the infinite `while` loop of the `main()` function, in order to increment `prev_ticks` every second:

```C
  /* USER CODE BEGIN WHILE */
  while (1)
  {
    new_ticks = HAL_GetTick() / PERIOD;
    if (new_ticks > prev_ticks)
      {
        prev_ticks = new_ticks;
      }
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
```

Start a debug session, and open the SWV settings window. In this window, tick **Enable** for the Comparator 0. Set **Var/Addr** to `prev_ticks`. Select **Write** for **Access** and **Data Value + PC** for **Generate**. Close the window.

![](images/variableTrace01.png)

Select **Window > Show View > SWV > SWV Trace Log**. This adds a new view, **SWV Trace Log** to the console tabbed notebook.

Now click on the SWV red button, to start the SWV session, and resume execution. 

The SWV Trace Log view displays every write operation to the variable, with the value of the PC for the instruction which performed the write operation:

![](images/variableTrace02.png)

Stop the execution. 

It's possible to add a graph view, which displays the evolution of the `prev_ticks` variable value over time. Select **Window > Show View > SWV > SWV Data Trace Timeline Graph**. This adds a new view, with the same name:

![](images/variableTimeline01.png)

#### Displaying a value with SWV

In addition to displaying the value of an existing variable, SWV can be used by the application to send some data to the debugging PC. This is done thanks to the `ITM_SendChar()` function, declared in `core_cm4.h` for our NUCLEO L476 board.

Let's use it to display `prev_ticks`, instead of enabling the Comparator 0. The main loop is modified as follows:

```C
  /* USER CODE BEGIN WHILE */
  while (1)
  {
    new_ticks = HAL_GetTick() / PERIOD;
    if (new_ticks > prev_ticks)
      {
        prev_ticks = new_ticks;
        // ITM_SendChar can send 32-bit values.
        ITM_SendChar(prev_ticks);
      }
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
```

In SWV settings window, untick **Enable** of Comparator 0, and tick port 0 in the **ITM Stimulus Ports**:

![](images/itmPort01.png)

ITM a is specific logic block, known as the Instrumentation Trace Macrocell, that enables applications to write arbitrary data to the SWO pin. It can do this over 32 multiplexed ports (or channels). The `ITM_SendChar()` function uses port 0. It's easy to add your own functions that write data to other ports.

Data is displayed in the SWV Trace Log view:

![](images/itmPort02.png)

## Enabling SWV for an existing application

To enable SWV for an existing application, we have to ensure that the three SWV pins are available, and we have to know the core clock frequency, as we saw above.

### Checking SWV pins availability

Of course, the first thing to do is to check the schematics of the development board you are using. Usually, SWDIO and SWCLK are available as they are used by SWD. It may happen that PB3 (SWO) is used for another function (GPIO, UART, SPI, etc.) If it is used by the code that you'd like to debug, sadly, you will not be able to use SWV. But if PB3 is used by a part of the code that you can ignore, for instance to control some peripherals that are not concerned by the problem you are debugging, chances are that you will be able to use SWO.

Let's consider that the PB3 pin is used by the application for another function, but that it is acceptable to switch it to its SWO function (of course, it must be wired to the JTAG probe. This is the case for the development board we are using here). Additionally, let's consider that we don't know well the source code of the application we are debugging. The question is: how to rapidly find where to configure PB3?

One way to answer this question is to set a watchpoint on the control register used to configure the related GPIO:

* Start a debug session
* If the SFR (Special Functions Registers) view is not displayed, open it with **Window > Show View > SFRs**
* Navigate to the MODER register for GPIOB:

![](images/sfrs01.png)

* The address of the register is displayed: `0x48000400`, in our case
* If the Memory view is not displayed, open it with **Window > Show View > Memory**
* Click on the `+` tool of the Memory view, and enter the above address:

![](images/memMonitor01.png)

* Right-click on the 32-bit value displayed for this address in the right-hand side pane, and select **Add Watchpoint (C/C++)...**:

![](images/addWatchpoint01.png)

* In the properties window, untick **Read** and tick **Write**
* Resume the execution. It will stop every time the control register is written. By looking at the calls chain in the Debug view, you will rapidly find the place where PB3 is configured, and you will be able to adapt the code so that PB3 is set to SWO

### Checking Core Clock frequency

It may happen that the application you are debugging has not been created with CubeMX. In which case, you don't have any `ioc` file, and you can't look at the graphical view of the clocks configuration.

In this case, one way to get the Core Clock frequency is to display the value of the `SystemCoreClock` global variable. This variable is defined in the `system_stm32l4xx.c` file, provided by the STM32CubeL4 package. In the Expressions view, add SystemCoreClock.

Start a debug session, and advance step by step until the clocks are configured. The value of SystemCoreClock displayed in the Expressions view should now be the one you can use to configure SWV:

![](images/clockExpr01.png)

## Reference documents

* [AN4989 - STM32 microcontroller debug toolbox](https://www.st.com/content/ccc/resource/technical/document/application_note/group0/3d/a5/0e/30/76/51/45/58/DM00354244/files/DM00354244.pdf/jcr:content/translations/en.DM00354244.pdf) - section 7.3
* [UM2609 - STM32CubeIDE user guide](https://www.st.com/content/ccc/resource/technical/document/user_manual/group1/f8/a2/48/77/68/e6/4b/74/DM00629856/files/DM00629856.pdf/jcr:content/translations/en.DM00629856.pdf) - section 4

