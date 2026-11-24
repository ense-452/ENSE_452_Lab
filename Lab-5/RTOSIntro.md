# ENSE 452 - Lab 5: RTOS Introduction

**Trevor Douglas**  
Software Systems Lab Instructor  
University of Regina

## Objective

The objective here is to implement FreeRTOS on our STM32 board. Using an embedded real-time system allows us to create tasks that will perform the functions of our application. We will create a simple task that blinks our green LED at a certain period. Another task will be responsible for receiving CLI commands and a third task for updating the UI.

## Reference Material

FreeRTOS is found on URCourses, as well as links to CMSIS-RTOS, which is maintained by Keil.

## Procedure

Create a new project for this lab as a subfolder of your repository. In the configuration file, make sure you:

1. Under System, make sure SYS is using a Timebase Source of TIM4.
2. Under Connectivity, enable USART2.
3. Under Middleware, enable FREERTOS.
4. Add tasks and message queues here as needed.

### Phase 1: Implement two tasks

Create two tasks that will blink the green LED at a certain rate. Implement these two tasks and observe the LED. Use the reference video on URCourses to complete this task.

### Phase 2: CLI Receive task

Modify the second task such that it will poll received UART transmissions. Use the blocking version of the HAL UART receive function.

One of the previous commands we created will change the duty cycle of the green LED. Implement that command to modify the period of the first task such that the period of the green LED will change.

Also, that first task that blinks the LED is our task that must meet a hardware deadline. What priority should that task have?

In order to update the LED task, you will need to send a message from the CLI Receive task to the LED task. This message will be the period to update the blinking. You can do this via code or by reloading the configuration `.ioc` file and adding queues to our tasks.

**Be careful! Using this method may overwrite your code.**

### Phase 3: CLI Update task

Create a third task that updates the screen like in previous labs. The current period of the green LED should be indicated in the status window.

## Things to think about

1. Task priority.
2. Task creation.
3. Sharing information between tasks.

## What To Submit

Commit and push your code to your repository.
