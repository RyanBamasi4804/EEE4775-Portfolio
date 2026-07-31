# App 3

https://wokwi.com/projects/468123070651717633

## Engineering Analysis Questions:


1. 
```
static void IRAM_ATTR gpio_isr_handler(void *arg)
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE; //this line stores whether the ISR woke 
                                                   //a higher priority task, required by FreeRTOS

    gpio_set_level(GPIO_NUM_19, 1);            //raises GPIO19

    xSemaphoreGiveFromISR(isrSemaphore,
                          &xHigherPriorityTaskWoken); // signals bottom half task of the work that needs to be performed.
    // or:
    // vTaskNotifyGiveFromISR(bottomHalfTaskHandle,
    //                        &xHigherPriorityTaskWoken);

    portYIELD_FROM_ISR(xHigherPriorityTaskWoken); //requests immediate context switch if higher priority task is ready
                                                  //without this line latency would increase

    gpio_set_level(GPIO_NUM_19, 0); //lowers GPIO19, marks end of ISR latency measurement
}


```

No I2C reads are in the ISR, no prints, no task delays, no loops, and no waiting on semaphores or mutexes.

2.  (4730) app3: [notif] press #1  latency=30 us (max=30) (notif count=1)
I (4730) app3: [sem] press #1  latency=2117 us (max=2117)

3. Comparing to the idle system the 4-load task, the 4-load tasks exhibits a slightly greater latency as the processor was executing higher priority tasks when the interrupt happened.

4. Commenting out portYIELD_FROM_ISR increased the latency between the interrupt and the task.fthe scheduler still loaded the semaphore but didn't immediately go to the bottom task, it waited a tick instead. Then the task was allowed to go.


---

## AI Disclosure

Me: how would you develop a concurrency diagram for this assignment? (with scaffold code)

CHATGPT:

The key idea is to show both task concurrency and interrupt concurrency.

What your diagram should contain
Core 0
+----------------------+
|    task_monitor      |
| (prints statistics)  |
+----------------------+
Core 1
Highest Priority
        │
        ▼
+----------------------+
| Task A (10 ms)       |
| Priority 14          |
+----------------------+
        │
        ▼
+----------------------+
| Task B (20 ms)       |
| Priority 10          |
+----------------------+
        │
        ▼
+----------------------+
| Task C (50 ms)       |
| Priority 5           |
+----------------------+
        │
        ▼
+----------------------+
| Task D (100 ms)      |
| Priority 2           |
+----------------------+
Add the interrupt path

This is the new part.

GPIO18 Button
      │
      │ Falling Edge
      ▼
+----------------------+
|   GPIO ISR           |
|  (IRAM_ATTR)         |
+----------------------+
      │
      │ xSemaphoreGiveFromISR()
      │
      ▼
+----------------------+
| Bottom-half Task     |
| (Semaphore Wait)     |
+----------------------+
      │
      │
      ▼
+----------------------+
| 3 ms I2C Read        |
| Logging              |
+----------------------+

Notice:

ISR is not inside the scheduler.
It interrupts whatever task is running.
It wakes the bottom-half task.
The bottom-half task performs the lengthy work.
Show the preemption

I would also add a note similar to this:

               Interrupt

Task A ---------------------------\
Task B ----------------------------> GPIO ISR
Task C ---------------------------/
Task D --------------------------/

ISR exits

↓

Bottom-half task becomes Ready

↓

Scheduler decides whether to switch

This demonstrates that the ISR can interrupt any running task.

Complete diagram

Something like this would earn full credit.

                      ESP32

                 +----------------+
                 | task_monitor   |
                 | Core 0         |
                 +----------------+

                        ||

=========================================================
                    Core 1 Scheduler
=========================================================

Priority 14
+-----------------------+
| Task A (10 ms)        |
+-----------------------+

Priority 10
+-----------------------+
| Task B (20 ms)        |
+-----------------------+

Priority 5
+-----------------------+
| Task C (50 ms)        |
+-----------------------+

Priority 2
+-----------------------+
| Task D (100 ms)       |
+-----------------------+

=========================================================

GPIO18 Button
      │
      ▼
+-----------------------+
| GPIO ISR (IRAM_ATTR)  |
+-----------------------+
      │
      │ xSemaphoreGiveFromISR()
      ▼
+-----------------------+
| Bottom-half Task      |
| waits on semaphore    |
+-----------------------+
      │
      ▼
+-----------------------+
| 3 ms I2C Read         |
| ESP_LOGI()            |
+-----------------------+
If you're making it in diagrams.net (draw.io)

I recommend using:

Rounded rectangles for FreeRTOS tasks.
A hexagon or octagon for the ISR to visually distinguish it from tasks.
A dashed arrow labeled Interrupt from the GPIO button to the ISR.
A solid arrow labeled xSemaphoreGiveFromISR() (or vTaskNotifyGiveFromISR()) from the ISR to the bottom-half task.
Place the ISR outside the scheduler area, since it is not scheduled like a normal task.
Keep the monitor task on Core 0 and the periodic tasks on Core 1, matching your implementation.