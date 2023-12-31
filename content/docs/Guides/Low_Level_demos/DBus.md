---
weight: 164
title: "DBus"
description: ""
icon: "article"
date: "2023-11-30T17:39:22-05:00"
lastmod: "2023-11-30T17:39:22-05:00"
draft: false
toc: true
---


uses irq and mostly taken from [basic uart](#basic-uart)  
[dbus decoding guide](https://drive.google.com/file/d/1a5kaTsDvG89KQwy3fkLVkxKaQJfJCsnu/view)

```cpp
/**
 * Copyright (c) 2020 Raspberry Pi (Trading) Ltd.
 *
 * SPDX-License-Identifier: BSD-3-Clause
 */

#include "pico/stdlib.h"
#include "hardware/uart.h"
#include "hardware/irq.h"
#include <stdio.h>
#include <iostream>

/// \tag::uart_advanced[]

// uart settings
#define UART_ID uart0
#define BAUD_RATE 100000
#define DATA_BITS 8
#define STOP_BITS 1
#define PARITY UART_PARITY_EVEN

// We are using pins 0 and 1, but see the GPIO function select table in the
// datasheet for information on which other pins can be used.
#define UART_TX_PIN 0
#define UART_RX_PIN 1

static int chars_rxed = 0;
static uint8_t data[18]{0};
static uint8_t save_data[18]{0};
uint32_t last_read = 0;

void parse_data()
{
    printf("\nBREAK: ");
    for (int i = 0; i < 18; i++)
    {
        printf("%x,", data[i]);
        save_data[i] = data[i];
        data[i] = 0;
    }
    printf("\n");
    chars_rxed = 0;
}

// RX interrupt handler
void on_uart_rx()
{
    if (time_us_32() - last_read > 300)
    {
        // std::cout << time_us_32() - last_read << std::endl;
        chars_rxed = 0;
    }
    while (uart_is_readable(UART_ID))
    {
        uint8_t ch = uart_getc(UART_ID);
        last_read = time_us_32();
        // printf("ch: %#x\n", ch);
        data[chars_rxed] = ch;
        chars_rxed++;

        if (chars_rxed >= 18)
            parse_data();

    }
}
int main()
{
    // Set up our UART with a basic baud rate.
    uart_init(UART_ID, BAUD_RATE); // 2400);

    // Set the TX and RX pins by using the function select on the GPIO
    // Set datasheet for more information on function select
    gpio_set_function(UART_TX_PIN, GPIO_FUNC_UART);
    gpio_set_function(UART_RX_PIN, GPIO_FUNC_UART);

    // Actually, we want a different speed
    // The call will return the actual baud rate selected, which will be as close as
    // possible to that requested
    // int __unused actual = uart_set_baudrate(UART_ID, BAUD_RATE);

    // Set UART flow control CTS/RTS, we don't want these, so turn them off
    uart_set_hw_flow(UART_ID, false, false);

    // Set our data format
    uart_set_format(UART_ID, DATA_BITS, STOP_BITS, PARITY);

    // Turn off FIFO's - we want to do this character by character
    uart_set_fifo_enabled(UART_ID, false);

    // Set up a RX interrupt
    // We need to set up the handler first
    // Select correct interrupt for the UART we are using
    int UART_IRQ = UART_ID == uart0 ? UART0_IRQ : UART1_IRQ;

    // And set up and enable the interrupt handlers
    irq_set_exclusive_handler(UART_IRQ, on_uart_rx);
    irq_set_enabled(UART_IRQ, true);

    // Now enable the UART to send interrupts - RX only
    uart_set_irq_enables(UART_ID, true, false);

    // OK, all set up.
    // Lets send a basic string out, and then run a loop and wait for RX interrupts
    // The handler will count them, but also reflect the incoming data back with a slight change!
    // uart_puts(UART_ID, "\nHello, uart interrupts\n");

    const uint LED_PIN = PICO_DEFAULT_LED_PIN;
    gpio_init(LED_PIN);
    gpio_set_dir(LED_PIN, GPIO_OUT);
    stdio_init_all();
    while (1)
    {

        printf(".");
        gpio_put(LED_PIN, 1);
        sleep_ms(250);
        gpio_put(LED_PIN, 0);
        sleep_ms(250);
    }
}


```