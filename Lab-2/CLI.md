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
## Embedded and Real-Time Software Systems - Laboratory 2: Command Line Interface

👨‍💻 SSE Lab Instructor: [Trevor Douglas](mailto:trevor.douglas@uregina.ca)

---
## Separating student code from CubeMX-generated code

Polling-based examples—no interrupt callbacks

---

## The generated handle is global—but access can remain controlled

Older CubeMX projects commonly define peripheral handles in `main.c`:

```c
/* Private variables -----------------------------------*/
TIM_HandleTypeDef htim4;
UART_HandleTypeDef huart2;
```

- The comment says “Private,” but the variables are not C-private.
- Without `static`, the linker can expose them through `extern`.
- The HAL needs persistent handle objects to store configuration and state.

> The design goal is not “no globals anywhere.” It is limited, explicit access.

---

## Keep generated ownership and inject dependencies

```text
CubeMX-generated main.c
  owns huart2
  initializes USART2
          │
          │  CLI_Init(&huart2)
          ▼
cli.c
  stores a private pointer
  calls UART HAL functions
```

This preserves both goals:

- CubeMX remains responsible for hardware configuration.
- Students use the real STM32 HAL from their own files.

---

## Organize by purpose—not one wrapper per peripheral

The module is named for what it does:

```text
cli.h       public CLI interface
cli.c       input, parsing, output, and HAL UART calls
main.c      generated setup plus dependency wiring
```

A CLI uses USART, but it is not a generic `StudentUSART` layer.

```c
CLI_Init(&huart2);
CLI_Task();
```

Students still work directly with `UART_HandleTypeDef`, `HAL_UART_Receive()`,
and `HAL_UART_Transmit()` inside the CLI implementation.

---

## `cli.h` exposes behavior and the required HAL dependency

```c
#ifndef CLI_H
#define CLI_H

#include "stm32f1xx_hal.h"

void CLI_Init(UART_HandleTypeDef *uart);
void CLI_Task(void);

#endif /* CLI_H */
```

- The header does not declare `extern UART_HandleTypeDef huart2`.
- The CLI can use any configured UART supplied by `main.c`.
- The dependency is visible at initialization time.

---

## `cli.c` keeps its UART pointer private

```c
#include "cli.h"
#include <string.h>

static UART_HandleTypeDef *cli_uart = NULL;

void CLI_Init(UART_HandleTypeDef *uart)
{
    cli_uart = uart;

    static const char message[] = "\r\nCLI ready\r\n> ";
    HAL_UART_Transmit(cli_uart, (uint8_t *)message,
                      sizeof(message) - 1U, 100U);
}
```

`cli_uart` has file scope, but `static` prevents other translation units from
accessing it directly.

---
## A polling task can receive and echo one character

```c
void CLI_Task(void)
{
    uint8_t ch;

    if (cli_uart == NULL) {
        return;
    }

    if (HAL_UART_Receive(cli_uart, &ch, 1U, 10U) == HAL_OK) {
        HAL_UART_Transmit(cli_uart, &ch, 1U, 100U);

        /* Add line buffering and command parsing here. */
    }
}
```

- The short receive timeout lets the main loop continue running.
- This introductory design is simple to trace in a debugger.
- Longer work should be divided across repeated `CLI_Task()` calls.

---

## Wire the module only inside CubeMX user-code blocks

In `main.c`, include the module:

```c
/* USER CODE BEGIN Includes */
#include "cli.h"
/* USER CODE END Includes */
```

Initialize after CubeMX initializes the USART:

```c
MX_USART2_UART_Init();

/* USER CODE BEGIN 2 */
CLI_Init(&huart2);
/* USER CODE END 2 */
```

---
Call it repeatedly from the generated main loop:

```c
while (1)
{
    /* USER CODE BEGIN 3 */
    CLI_Task();
    /* USER CODE END 3 */
}
```

---
