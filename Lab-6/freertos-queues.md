# CMSIS-RTOS2 Message Queues

*Using FreeRTOS through the CMSIS-RTOS2 API*

## Learning goals

By the end of this lab, you should be able to:

- Trace the order of messages through a queue.
- Create a queue and send and receive values.
- Explain what happens when a queue is full or empty.
- Design communication between two tasks.

## The communication problem

A sensor task produces these readings:

**21, 22, 23**

A display task runs less frequently.

**Discussion:** If both tasks use one shared variable, which readings might the display miss?

> **Notes:** Original motivating example. If each new reading overwrites the same variable before the display reads it, the display might see only 23. Sometimes that is acceptable. Ask whether the application needs every reading or only the latest reading.

**Discussion:**  What about global variables?

> **Notes:** Global variables are considered problematic because any code that can access them may depend on or change them, making a program harder to understand as it grows.
> - Hidden dependencies: a function’s behaviour depends on state that is > not visible in its arguments.
> - Unexpected changes: one part of the program can modify a value that another part assumes is stable.
> - Difficult debugging: when a value is wrong, you may need to inspect every place that writes it.
> - Difficult testing: tests must set up and reset global state, or one test can affect another.
> - Poor reuse: code tied to one global sensor value is harder to reuse for several sensors.
>- Concurrency problems: RTOS tasks can access the same data at overlapping times, requiring synchronization.


## A queue stores waiting messages

- A producer sends an item.
- The queue holds a copy until a consumer receives it.
- Receiving removes the item.
- All examples use message priority `0`, giving **FIFO** order.

| Operation | Queue contents, oldest first |
| --- | --- |
| Send 21, then 22 | 21, 22 |
| Receive one item | 22 |


## Queue capacity and item size

```c
osMessageQueueId_t readings;

readings = osMessageQueueNew(4, sizeof(int32_t), NULL);
if (readings == NULL) {
    /* Handle queue creation failure. Do not start users of this queue. */
}
```

- Capacity: **4 items**.
- Item size: **one `int32_t`**.
- Item storage: **16 bytes**, plus queue management overhead.

> **Notes:** Include stdint.h and cmsis_os2.h. NULL selects default queue attributes. These are snippets, not complete firmware. Initialize the kernel with osKernelInitialize(), create the queue and threads, then call osKernelStart(). Check creation results before starting users of the queue. In generated projects, use the existing initialization sequence.
> Source: https://arm-software.github.io/CMSIS_6/main/RTOS2/group__CMSIS__RTOS__KernelCtrl.html
> Source: https://arm-software.github.io/CMSIS_6/main/RTOS2/group__CMSIS__RTOS__Message.html
> The 16-byte calculation counts payload only, not management overhead or implementation-specific allocation.

## Timeouts use kernel ticks

```c
/* Convert short, bounded classroom delays, rounding up. */
static uint32_t ms_to_ticks(uint32_t ms) {
    return (uint32_t)(((uint64_t)ms * osKernelGetTickFreq()
                       + 999U) / 1000U);
}
```

- `0`: return immediately.
- A finite tick count: wait up to that interval.
- `osWaitForever`: wait without a timeout.

**100 ticks equals 100 ms only when the tick frequency is 1000 Hz.**

> **Notes:** This helper is defined once and used by later snippets. Call it after kernel initialization. Restrict inputs so the result fits a finite timeout below osWaitForever. Rounding up avoids converting a positive short interval to zero. Tick phase and scheduling affect observed wall-clock delay. This is application code, not a standard CMSIS conversion macro.
> Source: https://arm-software.github.io/CMSIS_6/main/RTOS2/group__CMSIS__RTOS__KernelCtrl.html

## Sending a value

```c
int32_t value = 21;

osStatus_t status = osMessageQueuePut(
    readings, &value, 0U, ms_to_ticks(100));
if (status != osOK) {
    /* Handle timeout or another reported error. */
}
```

| Argument | Meaning |
| --- | --- |
| `readings` | Destination queue |
| `&value` | Address of the item to copy |
| `0U` | Message priority used throughout this lesson |
| `ms_to_ticks(100)` | Maximum blocking time in kernel ticks |

> **Notes:** A send can return immediately when space is available. CMSIS defines message priorities, but the CMSIS-FreeRTOS adapter ignores msg_prio. Use zero consistently and do not rely on priority sorting with this backend.
> Source: https://arm-software.github.io/CMSIS-FreeRTOS/latest/page_technical_data.html

## Receiving a value

```c
int32_t received;

if (osMessageQueueGet(readings, &received, NULL,
                      ms_to_ticks(100)) == osOK) {
    /* Use received here. */
}
```

- `NULL` means no message-priority output is needed.
- The destination must have enough space for one item.
- Success copies and removes the oldest queued item.
- Use the received value only after a successful receive.

> **Notes:** Source: https://arm-software.github.io/CMSIS_6/main/RTOS2/group__CMSIS__RTOS__Message.html

## Full and empty queues

| Situation | Wait time is zero | Wait time is positive |
| --- | --- | --- |
| Send to a full queue | `osErrorResource` | Waits for space, then `osErrorTimeout` if still full |
| Receive from an empty queue | `osErrorResource` | Waits for data, then `osErrorTimeout` if still empty |

Successful puts and gets return `osOK`. A blocked thread waits without consuming CPU time.

**Discussion:** Why might blocking be better than repeatedly checking an empty queue?

> **Notes:** Answer: repeated polling can waste CPU time. A blocked task allows other ready tasks to run. Once unblocked, scheduling determines when it actually executes.
> Source: https://arm-software.github.io/CMSIS-FreeRTOS/latest/page_technical_data.html

## Prediction exercise

An empty queue holds **two integers**. Every operation uses zero wait time.

1. Send `10`.
2. Send `20`.
3. Send `30`.
4. Receive one item.
5. Send `40`.

**Which send fails? What remains in the queue?**

> **Notes:** Answer: sending 30 fails because the queue is full. The receive returns 10. Sending 40 succeeds. The remaining queue is 20, 40, oldest first. osMessageQueuePut does not overwrite an existing item when full.

## Copying values and pointers

For a queue created with `sizeof(int32_t)`:

```c
int32_t value = 21;
osMessageQueuePut(readings, &value, 0U, 0U);  // Assume success for this example.
value = 99;
```

The queued value remains **21**.

A queue of pointers copies the **address**. The application must keep the pointed-to data valid until the receiver finishes using it.

> **Notes:** Ask: What happens if a producer reuses a buffer before the consumer reads it? Answer: the pointer can lead to changed data. A queue protects its own operations, not arbitrary shared buffers.
> Source: https://www.digikey.com/en/maker/projects/introduction-to-rtos-solution-to-part-5-freertos-queue-example/72d2b361f7b94e0691d947c7c29a03c9

## Producer and consumer speeds

Suppose, on average:

- A producer generates **10 messages per second**.
- A consumer handles **6 messages per second**.

**Question:** What happens to a finite queue over time?

A larger queue can absorb a temporary burst. Sustained overload still requires a decision about waiting, dropping messages, or changing the processing rate.

> **Notes:** Original reasoning exercise. Before the queue fills, backlog grows by about 4 messages per second under these simplified assumptions. Once full, the sender's blocking and failure policy determines what happens. Do not imply a larger queue permanently fixes the throughput mismatch.

## Classroom lab: LED control

Adapted from the DigiKey companion challenge:

| Task | Responsibility |
| --- | --- |
| Serial task | Read a command such as `delay 250` and send the delay |
| LED task | Receive the delay and update LED timing |
| LED task | Send a status message after 100 blinks |
| Serial task | Receive and print the status message |

Use **two `osMessageQueueId_t` queues**, one for each direction.

> **Notes:** Source: https://www.digikey.com/en/maker/projects/introduction-to-rtos-solution-to-part-5-freertos-queue-example/72d2b361f7b94e0691d947c7c29a03c9
> Clarify the timing convention: if the LED stays on for 250 ms and off for 250 ms, one complete blink takes approximately 500 ms, plus execution overhead. Count one complete on/off cycle as one blink.

## Lab design decisions

Before coding, agree on:

- The data type carried by each queue.
- Queue capacities and send-failure behaviour.
- An allowed range for the delay command.
- How each task keeps doing its work while checking messages.

**Test cases:** a valid command, invalid input, rapid commands, and a full status queue.

> **Notes:** Suggested teaching choices: use a uint32_t delay and a status struct containing a blink count. Validate the delay before converting it to ticks. Avoid zero-tick delays. Keep the serial task responsive to both input and status messages. Do not let the LED task wait forever for a new command when it should continue blinking. Create threads with osThreadNew(). Use osMessageQueuePut()/osMessageQueueGet() for messages and osDelay(ms_to_ticks(delay_ms)) for LED waits. Poll for new LED commands with timeout zero between blink phases. osDelay takes ticks and is a thread-context call. These are extensions for classroom discussion, not claims about the video's exact implementation.

## Common mistakes

| Mistake | Better practice |
| --- | --- |
| Incorrect item size | Use `sizeof` with the actual message type |
| Ignored return value | Handle failed sends and receives |
| Milliseconds passed as ticks | Convert using `osKernelGetTickFreq()` |
| Pointer to expired data | Define data lifetime and ownership |
| Blocking inside an interrupt | Put/Get must use timeout `0` |

> **Notes:** Source: https://arm-software.github.io/CMSIS_6/main/RTOS2/group__CMSIS__RTOS__Message.html
> CMSIS-RTOS2 uses the same osMessageQueuePut/Get names in threads and interrupts. The adapter selects the backend operation. In an ISR, timeout must be zero and the interrupt priority must be allowed to call the RTOS. Do not create queues in an ISR. Keep the lab in thread context.

## Exit ticket

1. What does FIFO mean for our priority-zero messages?
2. What happens when a zero-wait send finds a full queue?
3. Why does changing the original integer leave its queued copy unchanged?
4. Why does the LED lab use two queues?

> **Notes:** Answers: 1. Items are received in their enqueue order. 2. The call fails immediately and adds no item. 3. The queue copied the integer's bytes when the send succeeded. 4. Commands and status messages travel in opposite directions with separate consumers.

## Sources and further practice

- [Original video: Introduction to RTOS Part 5 – Queue](https://www.youtube.com/watch?v=pHJ3lxOoWeI)
- [Shawn Hymel / DigiKey: companion explanation and challenge](https://www.digikey.com/en/maker/projects/introduction-to-rtos-solution-to-part-5-freertos-queue-example/72d2b361f7b94e0691d947c7c29a03c9)
- [CMSIS-RTOS2 kernel initialization and tick frequency](https://arm-software.github.io/CMSIS_6/main/RTOS2/group__CMSIS__RTOS__KernelCtrl.html)
- [CMSIS-RTOS2 message queue API](https://arm-software.github.io/CMSIS_6/main/RTOS2/group__CMSIS__RTOS__Message.html)
- [CMSIS-FreeRTOS adapter behaviour](https://arm-software.github.io/CMSIS-FreeRTOS/latest/page_technical_data.html)

**Extension:** Explain how your design should change if only the newest sensor reading matters.
