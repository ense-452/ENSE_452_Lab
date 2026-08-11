# Lab-1 Assignment

## Procedure

### Accept the assignment
On URCourses you will find a link to a GitHub classroom assignment.  Accept this assignemnt and make sure you select your User ID from the list provided so you submit under your username.  You must accept this assignment on your assigned lab day.

### Phase 1
- Blinky!  (The hello world of embedded)
- Start the STM32CubeIDE software and create a new STM32 project.
- Search and Select our Nucleo-64 board.
- Initiate all peripherals with their default mode.
- Use the graphical tool to ensure that PA5 is configured to connect to the Green LED on the board.
- Save the project and generate all the setup code.
- Look at the HAL documentation (20.2.4) to toggle this light and delay (6.1.3) for 1 second.  Make use of the HAL_Delay function.
- Make sure you check in your code and then push to your remote repository.  Use appropriate commit messages.
- 

### Phase 2
Now that you can control the light you will get rid of this blocking delay function and use timers.

- Look at the Clock tree (2.1 - Figure 1 and Figure 2 of the STM32F103RB data sheet) and notice that the clock source for Timer 4 is the APB1 clock.
- Open the *.ioc file that graphically allows you to configure the H/W for your board.  
- Under the Pinout and Configuration Tab open the Timers section for the Categories
- Enable Timer 4 by selecting the Internal Clock.  Notice that you can configure the parameters for the clock below in the Parameter Settings such as prescaler etc.
- Now Open the Clock Configuration tab (Up above beside the Pinout & Configuration tab).  Notice the clock speed for APB1.
- Look at the code generated in the configuration to understand how to update the prescaler and timer period.
- Consult the HAL documentation to get the timer started and to read it's current value.  HAL_TIM_Base_Start() and the macro HAL_TIM_GET_COUNTER().  You need to pass in the handler for this timer which is given to you when you generate the code.
- Based on this, set the pre-scaler to any desired number and get the light to blink every 1 second.

### Submission
Make sure you have commited your latest solution and pushed it to your repository for grading.