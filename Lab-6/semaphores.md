# CMSIS-RTOS2 Semaphores


## Learning goals

By the end of this lesson, you should be able to:

- Trace semaphore tokens as tasks signal and wait.
- Choose between a semaphore, mutex, and message queue.
- Create and use binary and counting semaphores through CMSIS-RTOS2.
- Explain how synchronization protects task-argument lifetimes.
- Design a bounded producer–consumer buffer.

## The synchronization problem

A worker needs a configuration value before it can begin. Its creator passes a pointer to a local variable, then changes that variable for the next worker.

**Discussion:** What happens if the worker has not read the value yet?

> **Notes:** The worker might read the next value. If the creator returns, the pointer might refer to an object whose lifetime has ended. Passing an address does not copy its contents. A handshake can establish when the creator may safely reuse the object.

## A semaphore counts permissions or events

A semaphore maintains a bounded token count. Acquiring consumes a token; releasing supplies one. When none are available, a task can wait. Semaphore operations coordinate the count atomically.

A semaphore does not identify which buffer or peripheral is available. That requires an additional allocation or data-management scheme. Semaphores are also useful for signaling between tasks. These concepts follow the [creator’s companion explanation](https://www.digikey.si/en/maker/projects/introduction-to-rtos-solution-to-part-7-freertos-semaphore-example/51aa8660524c4daba38cba7c2f5baba7).

## Binary and counting semaphores

| Purpose | Maximum | Initial count |
| --- | --- | --- |
| One pending notification | 1 | 0 |
| Three available resource permits | 3 | 3 |
| Up to five pending notifications | 5 | 0 |

Choose the initial count to represent what is available **now**. A notification should not start available if the event has not happened.

## CMSIS-RTOS2 operations

Include `cmsis_os2.h`. These examples use CMSIS-RTOS2, not CMSIS-RTOS v1.

```c
osSemaphoreId_t ready = osSemaphoreNew(1U, 0U, NULL);
if (ready == NULL) {
    /* Stop initialization and handle allocation failure. */
}
```

| Operation | Meaning |
| --- | --- |
| `osSemaphoreNew(max, initial, NULL)` | Create; returns `NULL` on failure |
| `osSemaphoreAcquire(id, timeout)` | Consume one token |
| `osSemaphoreRelease(id)` | Supply one token |

Successful operations return `osOK`. Acquisition with timeout `0` returns immediately: no token means `osErrorResource`. A finite timeout uses kernel ticks and may return `osErrorTimeout`. `osWaitForever` waits indefinitely. Releasing at the maximum count returns `osErrorResource`. See [ARM’s semaphore API](https://arm-software.github.io/CMSIS_6/main/RTOS2/group__CMSIS__RTOS__SemaphoreMgmt.html).

## Prediction exercise

A semaphore starts at zero and has a maximum of two. No other task accesses it. Every acquire uses timeout zero.

| Operation | Result | Count afterward |
| --- | --- | --- |
| Release | Success | 1 |
| Release | Success | 2 |
| Release | Cannot add another token | 2 |
| Acquire | Success | 1 |
| Acquire | Success | 0 |
| Acquire | No token available | 0 |

**Discussion:** Would a binary semaphore remember three events that arrived before any acquire?

> **Notes:** It can retain only one pending token. Counting semaphores also have a finite limit. Choose explicitly whether events may merge, whether overflow is an error, or whether each event needs a queued payload.

## Semaphore, mutex, or queue?

| Need | Suitable object |
| --- | --- |
| Signal that an operation happened | Semaphore |
| Limit concurrent use of a resource pool | Counting semaphore plus resource allocation |
| Protect shared state while modifying it | Mutex |
| Send individual values between tasks | Message queue |

A mutex has an owner: the acquiring thread must release it. A semaphore can be released by a different task, making it appropriate for a handshake. CMSIS mutex functions cannot run in interrupts. See [ARM’s mutex documentation](https://arm-software.github.io/CMSIS_6/main/RTOS2/group__CMSIS__RTOS__MutexMgmt.html).

Mutexes can provide priority inheritance; semaphores do not provide mutex ownership or that protection. The CMSIS-FreeRTOS adapter enables mutex priority inheritance by default. See [adapter behaviour](https://arm-software.github.io/CMSIS-FreeRTOS/latest/page_technical_data.html).

## Example: acknowledge a copied argument

The following original example uses a semaphore to tell a creator that a worker has finished copying its argument. The creator must itself run as a thread after the scheduler starts.

```c
#include "cmsis_os2.h"
#include <stdint.h>
#include <stddef.h>

/* Application hooks: implement these for your board. */
_Noreturn void App_Fatal(void);
void ProcessConfiguration(int32_t value);

typedef struct {
    int32_t value;
    osSemaphoreId_t copied;
} StartArgs;

static void Worker(void *argument)
{
    StartArgs local = *(const StartArgs *)argument;

    if (osSemaphoreRelease(local.copied) != osOK) {
        App_Fatal();
    }

    /* Use only the local copy after signaling. */
    ProcessConfiguration(local.value);
    osThreadExit();
}

static void Creator(void *argument)
{
    (void)argument;
    osSemaphoreId_t copied = osSemaphoreNew(1U, 0U, NULL);
    if (copied == NULL) {
        App_Fatal();
    }

    StartArgs args = { .value = 21, .copied = copied };
    if (osThreadNew(Worker, &args, NULL) == NULL) {
        App_Fatal();
    }

    if (osSemaphoreAcquire(copied, osWaitForever) != osOK) {
        App_Fatal();
    }

    /* Worker no longer reads args: reuse is now safe. */
    args.value = 99;

    /* Keep copied alive for the application's lifetime in this demo. */
    osThreadExit();
}
```

**Discussion:** Does the acknowledgment mean `ProcessConfiguration()` has finished?

> **Notes:** No. It means the argument was copied. Moving the signal changes what the handshake promises. This example deliberately retains the semaphore; a reusable production design should define object lifetime and cleanup. `App_Fatal()` must not return.

CMSIS passes the `argument` pointer to a new thread. Thread functions use the `void (*)(void *)` signature. For the FreeRTOS backend, finish a one-shot thread with `osThreadExit()` instead of returning. See [thread management](https://arm-software.github.io/CMSIS_6/main/RTOS2/group__CMSIS__RTOS__ThreadMgmt.html) and [backend limitations](https://arm-software.github.io/CMSIS-FreeRTOS/latest/page_technical_data.html).

## Starting the example

Integrate these steps into the board’s existing startup sequence:

```c
/* After board initialization, before the scheduler starts. */
if (osKernelInitialize() != osOK) {
    App_Fatal();
}
if (osThreadNew(Creator, NULL, NULL) == NULL) {
    App_Fatal();
}
if (osKernelStart() != osOK) {
    App_Fatal();
}
```

Do not initialize or start the kernel twice in a generated project. The creator’s blocking wait belongs inside its running thread. See [kernel initialization](https://arm-software.github.io/CMSIS_6/main/RTOS2/group__CMSIS__RTOS__KernelCtrl.html).

These are integration snippets, not complete board firmware. Thread stack sizes, heap configuration, and application hooks must suit the target.

## Classroom lab: a shared circular buffer

Adapt the creator’s challenge: five producers each write their identifier three times; two consumers remove and print values. Use a five-element circular buffer. Each identifier from 0 through 4 should appear exactly three times. For this exercise, implement the synchronization with two semaphores and a mutex before comparing it with a queue. See the [original challenge](https://www.digikey.si/en/maker/projects/introduction-to-rtos-solution-to-part-7-freertos-semaphore-example/51aa8660524c4daba38cba7c2f5baba7).

Create the synchronization objects before starting their users:

```c
osSemaphoreId_t empty = osSemaphoreNew(5U, 5U, NULL);
osSemaphoreId_t filled = osSemaphoreNew(5U, 0U, NULL);
osMutexId_t buffer_lock = osMutexNew(NULL);

if (empty == NULL || filled == NULL || buffer_lock == NULL) {
    App_Fatal();
}
```

The following is algorithm notation, not compilable C. Check every CMSIS call in the implementation.

| Producer sequence | Consumer sequence |
| --- | --- |
| Acquire `empty` | Acquire `filled` |
| Acquire `buffer_lock` | Acquire `buffer_lock` |
| Write at head; advance head | Copy from tail; advance tail |
| Release `buffer_lock` | Release `buffer_lock` |
| Release `filled` | Release `empty` |
| Repeat | Print the local copy; repeat |

**Discussion:** Why acquire the semaphore before the mutex?

> **Notes:** Holding the buffer mutex while waiting for space can prevent a consumer from freeing space. Holding it while waiting for data can prevent a producer from supplying data. Both can deadlock the design.

Protect console output separately if the logging implementation is not thread-safe. Keep slow printing outside the buffer lock. Give each producer stable argument storage, or acknowledge its argument copy before reusing storage.

Test with a slow consumer, a slow producer, and a one-element buffer. Check for missing values, duplicates, and out-of-range indices. At rest, with no operations in progress, empty and filled counts should sum to capacity; temporarily reserved slots complicate that check during execution.

## Connection to queues and global variables

A semaphore signals availability; it does not carry the reading itself. A shared buffer still needs clear ownership and protection.

For ordinary value transfer, a CMSIS message queue combines storage with synchronized send and receive operations:

```c
osMessageQueueId_t readings = osMessageQueueNew(5U, sizeof(int32_t), NULL);
```

Use `osMessageQueuePut()` and `osMessageQueueGet()` to copy values between tasks. Check creation and operation results. This can replace application-managed buffer indices and synchronization objects. See [CMSIS message queues](https://arm-software.github.io/CMSIS_6/main/RTOS2/group__CMSIS__RTOS__Message.html).

Passing a context pointer to each task can keep handles and shared state inside a module instead of exposing global variables everywhere. Moving data out of global scope does not, by itself, make concurrent access safe.

## Interrupts and timeouts

A CMSIS semaphore release may run in an ISR; an ISR acquire must use timeout zero. Create and delete semaphores outside interrupts. The interrupt priority must also satisfy the underlying FreeRTOS port’s RTOS-call restrictions. Keep the lab in thread context. See [semaphore ISR restrictions](https://arm-software.github.io/CMSIS_6/main/RTOS2/group__CMSIS__RTOS__SemaphoreMgmt.html).

Timeouts are ticks, not automatically milliseconds. For a short classroom interval, convert using the kernel frequency, round up, and keep the result below `osWaitForever`:

```c
static uint32_t ms_to_ticks(uint32_t ms)
{
    return (uint32_t)(((uint64_t)ms * osKernelGetTickFreq()
                       + 999U) / 1000U);
}
```

Call this after kernel initialization. Scheduling and tick phase affect observed elapsed time. See [kernel tick frequency](https://arm-software.github.io/CMSIS_6/main/RTOS2/group__CMSIS__RTOS__KernelCtrl.html).

## Exit ticket

1. Why does an event semaphore normally start at zero?
2. What does a successful acquire do to the count?
3. Why can a semaphore coordinate two different tasks while a mutex has an owner?
4. Why does a circular buffer need both availability tracking and mutual exclusion?
5. When would a message queue simplify the design?

> **Answers:** 1. No event has happened yet. 2. It consumes one token. 3. A semaphore permits one task to signal another; mutex release belongs to its owner. 4. Space/data counts do not serialize index and buffer updates. 5. When tasks need to transfer individual data items through bounded storage.
