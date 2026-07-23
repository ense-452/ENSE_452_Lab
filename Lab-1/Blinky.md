---
marp: true
paginate: true
title: ENSE 452 Lab 1
author: Trevor Douglas
theme: gaia
style: |
  section {
    background: #0B1220;
    color: #E6EDF3;
    font-size: 28px;
    text-align: left;
  }

  section:not(.lead) h1 {
    text-align: center;
  }

  h1, h2 {
    color: #7CC7FF;
  }

  blockquote {
    border-left: 6px solid #4DA3FF;
    color: #DCEBFF;
  }
---
<!-- _class: lead -->

# ENSE 452
## Embedded and Real-Time Software Systems - Laboratory

👨‍💻 SSE Lab Instructor: [Trevor Douglas](mailto:trevor.douglas@uregina.ca)

---

## Lab 1: Introduction to the hardware and software tools

### Pre reqs
What you know.
- Familiarity with Nucleo-64 Board or the STM32F1X series
- C/C++
- Keil µVision

What possibly is new?
- Git /GitHub
- RTOS
- Real time requirements

---

## Documentation

Many documents are available on urcourses, under Class Resources in the ARM STMicro Docs folder.  Some of the most useful are listed below. There may be updated versions of these documents available online, which you are welcome to substitute.

- STM32CubeIDEUserManual: UM2609
- STM32 Programmer's Manual: PM0056
- STM32 Reference Manual: RM0008
- Nucleo Board Schematic: MB1136
- STM32 HAL User Manual: UM1850
- STM32F103RB Data Sheet

---

## Equipment and Software Requirements}

Our target board is the Nucleo-64, and it has a lot of fun peripherals. The brain of this board is a STMicro STM32F103RB microcontroller, with 128 KiB of Flash, and 20 KiB of RAM and a 72MHz clock.

- PC for development.
- STM32CubeIDE Software (I am using Windows in the lab).
- Nucleo-64 development kit and USB Serial cable.
- Git (Git client or use command line).

---
## Objective

The objective of this lab is to get you started using the STM32CubeIDE and program the Nucleo-64 board to understand how to develop software for this target. You will also put your project under source control (revision control). First you'll get the LED on the board to blink at a certain rate. As you build your code make sure to commit changes and push to your repository as that is where your code will be evaluated.

---
## Procedure

### Phase 1
Blinky!  (The hello world of embedded)

- Demonstration !!
- Also create an account on your favorite GIT hosting site (bitbucket or github).  For instance you may name your repository enel452YourName''.  Make sure this repo is private and that you allow the professor, myself and TA's access.
- Clone your repository and put all your labs in a subdirectory and you can submit all your assignments under an assignments subdirectory.  Projects in this class will also have their own subdirectory.
- Start the STM32CubeIDE software and create a new STM32 project.
- Search and Select our Nucleo-64 board.
- Initiate all peripherals with their default mode.
- Use the graphical tool to ensure that PA5 is configured to connect to the Green LED on the board.
- Save the project and generate all the setup code.
- Look at the HAL documentation (20.2.4) to toggle this light and delay (6.1.3) for 1 second.  Make use of the HAL_Delay function.
- Make sure you check in your code and then push to your remote repository.  Use appropriate commit messages.
- 
---
### Phase 2
Now that you can control the light you will get rid of this blocking delay function and use timers.

- Look at the Clock tree (2.1 - Figure 1 and Figure 2 of the STM32F103RB data sheet) and notice that the clock source for Timer 4 is the APB1 clock.
- Open the *.ioc file that graphically allows you to configure the H/W for your board.  
- Under the Pinout and Configuration Tab open the Timers section for the Categories
- Enable Timer 4 by selecting the Internal Clock.  Notice that you can configure the parameters for the clock below in the Parameter Settings such as prescaler etc.
- Now Open the Clock Configuration tab (Up above beside the Pinout & Configuration tab).  Notice the clock speed for APB1.
- Consult the HAL documentation to get the timer started and to read it's current value.  HAL_TIM_Base_Start() and the macro HAL_TIM_GET_COUNTER().  You need to pass in the handler for this timer which is given to you when you generate the code.
- Based on this, set the pre-scaler to any desired number and get the light to blink every 1 second.

### What To Submit
On URCourses put the link to your git repository in the submission link.  The TA will checkout the version (date) submitted at the
deadline.  