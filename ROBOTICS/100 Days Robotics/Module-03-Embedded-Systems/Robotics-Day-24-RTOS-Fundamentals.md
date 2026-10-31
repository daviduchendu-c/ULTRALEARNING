# Day 24: RTOS Fundamentals

*100 Days of Robotics Engineering, Module 3: Embedded Systems*

**Time:** about 4 hours

**Level:** advanced beginner to intermediate. You need Day 19 (interrupts and shared data), Day 23 (the superloop and the state machine) and a working PlatformIO or Arduino IDE setup from earlier days. Today the target is an ESP32, not the Uno, because the Uno's 2 KB of SRAM is too small for a real multitasking demonstration.

**By the end of this day, you will have:**

- Two FreeRTOS tasks running on an ESP32, pinned to separate cores, passing samples through a queue
- A mutex-protected counter shared between tasks, with its behavior compared with an unprotected version
- A race-condition demonstration that loses updates, with the logged numbers that prove it
- A short note on priority inversion and deadlock, with one example of each in your own words

**What you need:**

- An ESP32 DevKit board with a 240 MHz dual-core ESP32 and 4 MB of flash, about $6 to $12
- A USB cable that carries data, not only power, about $3
- PlatformIO Core 6.1.x with the VS Code extension, or PlatformIO Core alone from the command line. Both are free and open source. The Espressif platform for PlatformIO is also free.
- The Arduino framework for ESP32 (Arduino-ESP32 core 3.x), installed automatically by PlatformIO as part of the `espressif32` platform. Check the build log for the core version you get.
- A potentiometer, 10 kOhm, about $1, on GPIO 34
- One LED and 220 ohm resistor on GPIO 2 or another output pin, about $0.20
- A breadboard and jumper wires, about $5
- Optional: a logic analyzer, about $10 to $20, for the timing of the tasks

**Prerequisites:**

- Day 16: the difference between a sketch and its build output
- Day 19: interrupts and shared data
- Day 23: the superloop, the state machine and the telemetry frame
- A basic understanding of how a CPU runs more than one program at a time, from an operating systems course or equivalent

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Tasks and the scheduler](#tasks-and-the-scheduler)
- [Priorities and preemption](#priorities-and-preemption)
- [Queues](#queues)
- [Semaphores and mutexes](#semaphores-and-mutexes)
- [Timing with delays and the tick](#timing-with-delays-and-the-tick)
- [Concurrency and race conditions](#concurrency-and-race-conditions)
- [Priority inversion and deadlock](#priority-inversion-and-deadlock)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a two-task sensor pipeline and a race demo](#project-a-two-task-sensor-pipeline-and-a-race-demo)
- [Check your understanding](#check-your-understanding)
- [Answers](#answers)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

A superloop from Day 23 runs one task after another, and each task must finish quickly so the next one can run. That design is simple, and it works well for a logger that samples a sensor every 200 ms. It becomes hard to manage when the system has many jobs with different deadlines. A motor control loop that must run every millisecond, a radio that must respond within 10 ms and a display that updates every half second all compete for the same loop. Every time one job gets slower, the others miss their timing.

A real-time operating system (RTOS) solves that problem by giving each job its own task, with its own stack and its own priority. The scheduler decides which task runs, and a higher-priority task can interrupt a lower one at any time. Tasks communicate through queues, semaphores and mutexes instead of shared globals. The result is a system where each job is written as if it runs alone, and the RTOS handles the interleaving.

The price of that power is a new set of bugs. Two tasks that share a variable without protection can lose updates. A mutex held by a low-priority task can block a high-priority task for a long time. A pair of tasks that take two locks in opposite orders can freeze forever. Today you will build a small pipeline with two tasks, then break it on purpose, so that you can recognize these bugs in your own code.

---

## Short history

| Period | What happened | Why it matters for robots today |
|---|---|---|
| 1960s | Multiprogramming and time-sharing systems introduce the idea of switching between programs, which becomes the basis of scheduling. | The scheduler in an RTOS is a small, fast version of the same idea. |
| 1970s to 1980s | Real-time kernels, such as those used in aerospace and industrial control, add fixed priorities and deterministic response times. | Hard real-time requirements in robotics come from this line of work. |
| 1990s | Open-source kernels, including early versions of FreeRTOS, make the approach available to hobbyists and small companies. | FreeRTOS is now used on millions of devices, including the ESP32 family. |
| 2000s | Priority inheritance is widely adopted for mutexes after the Mars Pathfinder anomaly of 1997 shows how priority inversion can reset a spacecraft. | Mutexes in FreeRTOS support priority inheritance, which prevents that failure. |
| Today | FreeRTOS, Zephyr and other RTOS kernels run on microcontrollers that control drones, arms and vehicles. | ESP32 robotics projects often use FreeRTOS, even when the code calls Arduino functions. |

The Pathfinder anomaly is one of the clearest examples of why scheduling matters. The fix was a configuration change that enabled priority inheritance on a mutex, and many real-time courses cover it.

---

## Glossary

- **RTOS (real-time operating system):** a small operating system that schedules tasks with bounded timing.
- **Task:** a function that runs in its own stack and that the scheduler can start, stop and resume.
- **Stack:** the memory a task uses for function calls and local variables.
- **Scheduler:** the part of the RTOS that picks the next task to run.
- **Priority:** a number that tells the scheduler which task to run when more than one is ready. In FreeRTOS, a higher number means a higher priority.
- **Preemption:** the scheduler stopping a running task so that a higher-priority task can run.
- **Tick:** the regular interval at which the scheduler checks for work. On ESP32 builds, it is commonly 1 ms.
- **Queue:** a first-in, first-out buffer of fixed-size items that tasks use to pass data.
- **Semaphore:** a counter used for signaling between tasks or for limiting access to a resource.
- **Binary semaphore:** a semaphore with a count of 0 or 1, used for signaling.
- **Mutex (mutual exclusion):** a lock that only one task can hold at a time, with priority inheritance in FreeRTOS.
- **Priority inversion:** a situation where a high-priority task waits for a low-priority task that holds a lock, and a medium-priority task runs in the meantime.
- **Priority inheritance:** a rule that temporarily raises the priority of the lock holder to the highest priority waiting for the lock.
- **Deadlock:** a state where two or more tasks wait for each other's locks, and none can continue.
- **Race condition:** a bug where the result depends on the order in which tasks run.

---

### Tasks and the scheduler

A kitchen with one cook and a long list of orders can run each order in turn, but a long order delays every other one. A kitchen with several cooks, each with a station and a priority for each order, can run many orders at once. A task is one cook with one station. The scheduler is the head chef who decides which cook works at each moment.

From first principles, a task is a function that never returns. It runs a loop, does its work and waits for something before running again. Each task has its own stack, so its local variables do not interfere with those of other tasks. On the ESP32, FreeRTOS tasks are created with `xTaskCreatePinnedToCore`, which takes the task function, a name, a stack size in bytes, a parameter, a priority, an optional handle and the core number, where 0 or 1 pins the task to that core.

Consider the stack size as a worked example. A task that calls `Serial.printf` may need several kilobytes of stack for the formatting code. A stack that is too small overflows, and the result is a crash or a silent corruption. The common starting value of 4096 bytes is a reasonable first guess, and the FreeRTOS function `uxTaskGetStackHighWaterMark` tells you how much of the stack was left unused at its lowest point. A high-water mark of 500 bytes means the stack is safe with a small margin, and zero means the task has already overflowed.

The common mistake is to assume that tasks run in parallel on a single core. On a single core, the scheduler gives each task a share of the CPU. On a dual-core ESP32, two tasks pinned to different cores can run at the same instant. That is why the race condition in this lesson is easy to show on an ESP32 and hard to show on a single core.

On the job, a task list with stack sizes and priorities is part of every RTOS design. A reviewer checks it for stack margins and for the rule that each task has one owner.

---

### Priorities and preemption

A hospital emergency room treats the most serious patient first, and a patient with a minor injury waits. The triage decision is a priority. The RTOS scheduler works the same way. When a task with a higher priority becomes ready, the scheduler stops the running task and switches to the higher one. The lower task resumes later, with its state saved.

From first principles, FreeRTOS uses fixed-priority preemptive scheduling. Among the ready tasks, the highest priority runs. Tasks with the same priority share the CPU in round-robin order, using a time slice set by the tick. The idle task runs at priority 0 and runs only when no other task is ready. A task that never waits, such as one with a tight loop and no delay, starves all lower-priority tasks, which is a common bug.

The worked example uses two tasks: a sensor task at priority 2 that samples every 100 ms, and a logger task at priority 1 that prints the data. When the sensor task is ready, it preempts the logger, reads the sensor and goes back to sleep. The logger then resumes. The sensor sample time is almost exact, because the scheduler gives the sensor task the CPU as soon as its time arrives, even if the logger is printing.

The common mistake is to give every task the same priority and assume the timing will work. With equal priorities, a long print in one task delays the sample in another, because the scheduler only switches at the time slice. Set priorities by deadline: the task with the tightest timing gets the highest priority.

On the job, priority assignment is a design review item. Engineers write a table of tasks, priorities and deadlines, and they test the table with the worst-case load.

---

### Queues

A post office with a sorting table and a mail carrier moves packages from one station to another without the two people talking directly. The table holds the packages in order, and each person takes one from the table when ready. A FreeRTOS queue does the same for tasks. The producer puts an item in, the consumer takes it out, and neither needs to know when the other runs.

From first principles, a queue holds a fixed number of items of a fixed size. A send copies the item into the queue, and a receive copies it out. The copy means the sender can reuse its variable right away. A send to a full queue fails or waits, depending on the timeout. A receive from an empty queue waits, depending on the timeout. The waits let the scheduler run other tasks, which is why queues are the normal way to pass data between tasks.

The worked example is the project's sample queue. It has eight slots and holds `Sample` structures. The struct holds a 32-bit time and a 16-bit reading, and the compiler pads it to 8 bytes for alignment, so the queue uses 64 bytes in total. Use `sizeof(Sample)` in the code and do not assume the packed size. The sensor task sends with a zero timeout, so it never waits. If the logger is slow and the queue fills, the send fails, and the sensor task records a drop and moves on. A queue of eight slots at 100 ms per sample holds 800 ms of backlog, so a logger that stalls for longer than that begins to lose samples. The number of drops is a measurement of how far the consumer falls behind.

The common mistake is to use a queue as a shared variable, for example by sending the pointer to a global that both tasks then change. The queue copies the item, which is the point, and passing a pointer back to shared memory brings back the race conditions the queue was meant to prevent. The second mistake is a queue that is too small, which drops data under load.

On the job, queues carry sensor readings, motor commands and telemetry between tasks in most RTOS designs. Their depth and their overflow policy are a design decision that should be written down.

---

### Semaphores and mutexes

Two people who want to use one printer share a sign-up sheet, and only one person can use it at a time. A mutex is the sign-up sheet. A task takes it, uses the shared resource and gives it back. A semaphore is a counter, such as a set of parking spaces, where a task takes a space to use the resource and gives it back when done. A binary semaphore is a single parking space used for signaling.

From first principles, `xSemaphoreTake` tries to decrement the count, and it waits if the count is zero, up to a timeout. `xSemaphoreGive` increments the count and wakes a waiting task. A mutex in FreeRTOS is a binary semaphore with an owner. Only the task that took the mutex may give it back, and the mutex supports priority inheritance. A binary semaphore has no owner and no inheritance, so it is better for signaling than for protecting data.

The worked example is the statistics counter in the project. The sensor task increments `sampleCount`, and the logger reads it. Both operations take the mutex with `xSemaphoreTake(statsMutex, pdMS_TO_TICKS(10))`. The ten-millisecond timeout means the task gives up if the mutex is held for too long, and the code can record the failure instead of waiting forever. A failure rate above zero is a sign that a task holds the lock for too long, which is a design problem to fix.

The common mistake is to forget to give the mutex back on an error path. A function that returns early without `xSemaphoreGive` leaves the mutex held, and the next task that tries to take it waits until its timeout. The fix is to write a single exit path, or to use a small RAII helper in C++ that gives the mutex back when it goes out of scope.

On the job, mutexes protect shared hardware buses, shared configuration and shared logs. Locking discipline, including the order and the duration of each lock, is part of the firmware review.

---

### Timing with delays and the tick

A commuter who leaves the house at 7:00 every day arrives at the office on time, even if the walk takes a different amount of time on each day. A commuter who waits 30 minutes after each walk drifts later and later. RTOS timing has the same two forms. `vTaskDelay` waits a set time after the call, so the period drifts with the task's own work. `vTaskDelayUntil` waits until a set time measured from the last wake, so the period stays fixed.

From first principles, the RTOS keeps time with a tick count that increases at the tick rate. On many ESP32 builds the tick is 1 ms, and on others it is 10 ms, and the setting is `configTICK_RATE_HZ`. The macro `pdMS_TO_TICKS(100)` converts milliseconds to ticks correctly for the build, which is why it should be used instead of a number. A delay of 100 ticks is 100 ms on a 1 kHz tick and 1 second on a 100 Hz tick.

The worked example compares the two delay calls. A sensor task with `vTaskDelay(pdMS_TO_TICKS(100))` and a 3 ms read has a period of 103 ms, and after one hundred samples the drift is 300 ms. A sensor task with `vTaskDelayUntil` keeps a period of 100 ms, because it wakes at the scheduled times, and the drift is zero except for the time the scheduler takes to switch.

The common mistake is to assume that `vTaskDelay(1)` waits for one millisecond. On a 100 Hz tick it waits 10 ms, which is a common surprise for ESP-IDF users. Check `configTICK_RATE_HZ` in the build output and use `pdMS_TO_TICKS` for every delay.

On the job, timing requirements are written as periods and deadlines, and the RTOS delay type is chosen to match. A control task with a fixed period uses the periodic call, and a background task uses the simple delay.

---

### Concurrency and race conditions

Two people who edit the same spreadsheet cell at the same time can each overwrite the other's entry. A race condition in firmware is the same kind of error, and it appears only when the tasks run at the same moment or interleave in a specific order. The bug depends on timing, so it may appear once in a million runs and never on the bench you test with.

From first principles, an increment such as `count = count + 1` is three operations in the machine: load the value, add one and store it. If two tasks run that sequence on the same variable at the same time, both may load the same value, and one increment is lost. On a single core, the scheduler can switch between the load and the store, which also loses an update. On a dual-core ESP32, the two cores can execute the sequence at the same moment.

The worked example is the race demo in the project. Two tasks each increment a shared integer one million times, and the expected final result is two million. Without protection, the result is usually lower, and the difference is the number of lost updates. With a mutex around each increment, the result is exact, and the program takes longer. The gap between the two results is the cost of the protection and the evidence that the race exists.

The common mistake is to use `volatile` as protection. As in Day 19, `volatile` only stops the compiler from caching the variable. It does not make the increment atomic, and the race remains. The correct tools are a mutex, a critical section with `portENTER_CRITICAL` for very short operations, or an atomic type. For a multi-word structure, a mutex is the normal choice.

On the job, race conditions are one of the most expensive firmware bugs, because they are hard to reproduce. A design that gives each shared variable one owner, and passes data by queue, avoids most of them.

---

### Priority inversion and deadlock

A road with one lane and a broken-down car can delay everyone behind it. A broken-down car that is blocking the lane while a fire truck waits is a serious problem. Priority inversion is the RTOS version of that scene. A high-priority task waits for a mutex held by a low-priority task, and a medium-priority task runs in the meantime, because it is ready and the low-priority task is not. The high-priority task waits longer than either the low or the medium task would suggest.

From first principles, the fix is priority inheritance. When a high-priority task waits for a mutex, the holder's priority is raised to the waiting task's level until the holder gives the mutex back. The holder can then finish its work, and the high-priority task runs. FreeRTOS mutexes created with `xSemaphoreCreateMutex` support this behavior. Binary semaphores created with `xSemaphoreCreateBinary` do not, so they are the wrong tool for a shared resource.

Deadlock is a different problem. Two tasks each take one lock and then wait for the other's lock. Neither gives up, and both wait forever. The worked example is a task that takes mutex A and then mutex B, and a task that takes mutex B and then mutex A. The fix is a fixed lock order: every task takes A before B, and no task holds one while waiting for the other in the reverse order. A timeout on each take, as used in the project, turns a deadlock into a visible error.

The common mistake is to think that a timeout solves deadlock. A timeout only detects it, so the design still needs a fixed lock order, and the error path needs to release every lock it holds. The second mistake is to hold a mutex while calling a function that might block, such as a long print, which extends the lock and increases the chance of inversion.

On the job, priority inversion and deadlock are the two bugs that real-time reviewers look for first. A lock table with the order and the owner of each mutex is a standard part of an RTOS design.

---

## Hands-on exercises

1. **Install the toolchain.** Install PlatformIO Core 6.1.x, create a new project for the `esp32dev` board with the Arduino framework, and build the default blink program. Record the platform version and the Arduino-ESP32 core version from the build log.

2. **Check the tick rate.** Print `configTICK_RATE_HZ` from the build configuration, or check the tick setting in the build log. Then measure `vTaskDelay(pdMS_TO_TICKS(100))` with `millis()` over ten calls.

3. **Measure the stack.** Create a task with a 4096-byte stack that calls `Serial.printf` in a loop. Print `uxTaskGetStackHighWaterMark` after a minute. Record the value and say whether the margin is safe.

4. **Run the queue pipeline.** Build the project's sensor and logger tasks. Watch the output for 60 seconds and count the samples. Then add a 300 ms delay inside the logger and record the number of drops.

5. **Show the race.** Run the race demo without protection and record the final count. Run it again with the mutex and record the count. Compare the two results and the time taken.

6. **Make a deadlock.** Write two tasks that take two mutexes in opposite orders, with a 200 ms delay between each take. Observe the behavior. Then fix the order and confirm that the deadlock is gone.

---

## Project: a two-task sensor pipeline and a race demo

The goal is a repository that runs a sensor task and a logger task on an ESP32, passes samples through a queue, protects a shared counter with a mutex, and includes a race demo that shows lost updates without protection. The project uses the same ideas as the superloop from Day 23, but it is built on tasks instead of a loop.

**Repo name:** `100-days-robotics-day-24-rtos-fundamentals`

**Repo layout:**

```
100-days-robotics-day-24-rtos-fundamentals/
├── README.md
├── pipeline/
│   ├── platformio.ini
│   └── src/
│       └── main.cpp
├── race_demo/
│   ├── platformio.ini
│   └── src/
│       └── main.cpp
├── logs/
│   ├── pipeline_60s.csv
│   ├── pipeline_slow_logger.csv
│   ├── race_unprotected.txt
│   ├── race_protected.txt
│   └── stack_high_water.txt
├── docs/
│   ├── task_table.md
│   ├── lock_order.md
│   └── wiring.md
└── photos/
    └── bench.jpg
```

Make `pipeline` and `race_demo` two separate PlatformIO projects. Each folder gets its own `platformio.ini` at its root, and `pio run` in that folder builds that project only. This keeps the two programs apart, so the race demo never picks up the pipeline code.

Use this `platformio.ini` in both folders:

```ini
[env:esp32dev]
platform = espressif32        ; pin the version you install, see pio pkg list
board = esp32dev
framework = arduino
monitor_speed = 115200
```

**Starter code, `pipeline/src/main.cpp`:**

```cpp
#include <Arduino.h>

struct Sample {
  uint32_t ms;
  uint16_t raw;
};

static QueueHandle_t sampleQueue;
static SemaphoreHandle_t statsMutex;
static uint32_t sampleCount = 0;
static uint32_t dropCount = 0;
static const int POT_PIN = 34;               // ADC1 input-only pin
static const int LOGGER_DELAY_MS = 0;        // set to 300 for the slow-logger test

void sensorTask(void *arg) {
  TickType_t lastWake = xTaskGetTickCount();
  for (;;) {
    Sample s{millis(), (uint16_t)analogRead(POT_PIN)};
    if (xSemaphoreTake(statsMutex, pdMS_TO_TICKS(10)) == pdTRUE) {
      sampleCount++;
      xSemaphoreGive(statsMutex);
    }
    if (xQueueSend(sampleQueue, &s, 0) != pdTRUE) {
      if (xSemaphoreTake(statsMutex, pdMS_TO_TICKS(10)) == pdTRUE) {
        dropCount++;
        xSemaphoreGive(statsMutex);
      }
    }
    vTaskDelayUntil(&lastWake, pdMS_TO_TICKS(100));
  }
}

void loggerTask(void *arg) {
  Sample s;
  for (;;) {
    if (xQueueReceive(sampleQueue, &s, portMAX_DELAY) == pdTRUE) {
      uint32_t samples = 0, drops = 0;
      if (xSemaphoreTake(statsMutex, pdMS_TO_TICKS(10)) == pdTRUE) {
        samples = sampleCount;
        drops = dropCount;
        xSemaphoreGive(statsMutex);
      }
      Serial.printf("%lu,%u,%lu,%lu\n",
                    (unsigned long)s.ms, s.raw,
                    (unsigned long)samples, (unsigned long)drops);
      if (LOGGER_DELAY_MS > 0) vTaskDelay(pdMS_TO_TICKS(LOGGER_DELAY_MS));
    }
  }
}

void setup() {
  Serial.begin(115200);
  analogSetAttenuation(ADC_11db);
  sampleQueue = xQueueCreate(8, sizeof(Sample));
  statsMutex = xSemaphoreCreateMutex();
  xTaskCreatePinnedToCore(sensorTask, "sensor", 4096, nullptr, 2, nullptr, 1);
  xTaskCreatePinnedToCore(loggerTask, "logger", 4096, nullptr, 1, nullptr, 0);
}

void loop() {
  delay(1000);                               // the Arduino loop task has nothing to do
}
```

The `loop()` function runs in its own task. The pipeline does its work in the two tasks, so `loop()` only waits. The `analogSetAttenuation(ADC_11db)` call gives the ADC a range closer to 3.3 V, which is a choice to test on your bench. The ESP32's ADC is not linear across its range, so the readings are a relative measure, and the README should say so.

**Starter code, `race_demo/src/main.cpp`:**

```cpp
#include <Arduino.h>

static int sharedCount = 0;                  // unprotected in the first run
static SemaphoreHandle_t countMutex;
static SemaphoreHandle_t doneSem;
static const int N = 200000;
static const bool USE_MUTEX = false;         // set to true for the protected run

void incrementTask(void *arg) {
  for (int i = 0; i < N; i++) {
    if (USE_MUTEX) xSemaphoreTake(countMutex, portMAX_DELAY);
    sharedCount = sharedCount + 1;
    if (USE_MUTEX) xSemaphoreGive(countMutex);
  }
  xSemaphoreGive(doneSem);
  vTaskDelete(nullptr);
}

void setup() {
  Serial.begin(115200);
  countMutex = xSemaphoreCreateMutex();
  doneSem = xSemaphoreCreateCounting(2, 0);
  xTaskCreatePinnedToCore(incrementTask, "incA", 4096, nullptr, 1, nullptr, 0);
  xTaskCreatePinnedToCore(incrementTask, "incB", 4096, nullptr, 1, nullptr, 1);
}

void loop() {
  static bool printed = false;
  if (!printed) {
    xSemaphoreTake(doneSem, portMAX_DELAY);
    xSemaphoreTake(doneSem, portMAX_DELAY);
    Serial.printf("expected %d, got %d, lost %d\n",
                  2 * N, sharedCount, 2 * N - sharedCount);
    printed = true;
  }
  delay(1000);
}
```

The race demo's `sharedCount` is a plain integer. It is shared between two tasks on different cores, and the increment is not atomic. The mutex version takes the lock on every increment, which is slow, and it is the reference for the correct result. The `USE_MUTEX` flag is a compile-time choice, and the README should record which build produced each log.

**Step-by-step tasks:**

1. Create the repository and folder tree. Initialize git and commit the README skeleton.
2. Write `platformio.ini` with the two environments. Build `pipeline` and record the platform and core versions in `docs/task_table.md`.
3. Upload the pipeline with `LOGGER_DELAY_MS = 0`. Record 60 seconds of output in `logs/pipeline_60s.csv`. Count the drops at the end.
4. Set `LOGGER_DELAY_MS = 300`, upload again, and record 60 seconds in `logs/pipeline_slow_logger.csv`. Count the drops and compare with the first run.
5. Add a stack high-water check to the logger task using `uxTaskGetStackHighWaterMark(nullptr)` and log the result to `logs/stack_high_water.txt`.
6. Run the race demo with `USE_MUTEX = false`, and save the output to `logs/race_unprotected.txt`. Run it again with `USE_MUTEX = true`, and save the output to `logs/race_protected.txt`. Record the time for each run.
7. Write `docs/lock_order.md` with the mutexes in the project, the order in which they are taken, and a note on the deadlock test from the exercises.
8. Write the README with the checklist below.

**README checklist:**

- [ ] The platform and Arduino-ESP32 core versions are stated from the build log
- [ ] The task table lists each task, its priority, its core, its stack size and its high-water mark
- [ ] The queue depth and the drop count are reported for both logger speeds
- [ ] The race demo results are reported for both builds, with the lost count
- [ ] The lock order is documented
- [ ] The README explains the ADC is a relative measure on the ESP32, and why
- [ ] One paragraph explains priority inversion, in your own words, with a case from the project

**Done when:**

- The pipeline runs for 60 seconds with no drops at the normal logger speed, and the slow-logger run shows drops that match the queue depth math in the README
- The race demo's unprotected run shows lost updates, and the protected run shows zero lost updates
- The stack high-water marks are at least 200 bytes for every task, or the README explains a smaller margin and the fix
- The repository has at least three commits, and every number in the README matches a log file

**Evidence to commit to GitHub:** both `main.cpp` files, the `platformio.ini`, the logs, the task table, the lock order document, the wiring document, the bench photo and the README. Do not commit the `.pio` build folders.

This project is light on hardware and heavy on evidence. The lost-update count and the drop count are numbers a reviewer can reproduce, and they show that you understand the cost of sharing data and the cost of falling behind.
---

## Check your understanding

1. A task has priority 3 and another has priority 2. The priority 3 task is ready. Explain what the scheduler does and what happens to the priority 2 task.
2. A queue holds eight items, and the producer sends one item every 100 ms. The consumer stops for 1 second. Compute the number of items lost, if any, and explain whether the queue depth is enough.
3. A task uses `vTaskDelay(pdMS_TO_TICKS(100))` after a 5 ms read. Compute the drift after 1,000 samples, and say which call would prevent it.
4. Two tasks increment a shared counter without a mutex, on two cores. Explain why the final count is smaller than expected, and name the two fixes that remove the race.
5. A high-priority task waits for a mutex held by a low-priority task, while a medium-priority task runs. Name the problem, and the FreeRTOS feature that fixes it.
6. Task A takes lock X and then waits for lock Y. Task B takes lock Y and then waits for lock X. Name the problem and the rule that prevents it.

---

## Answers

1. The scheduler stops the running task and switches to the priority 3 task, because a higher number means a higher priority in FreeRTOS. The priority 2 task waits until the priority 3 task blocks or finishes, and then it resumes where it stopped.
2. The producer sends about 10 items during the 1 second stop. The queue holds eight, so about two items are lost, unless the consumer catches up. The depth is not enough for a 1 second stall at 10 items per second. A deeper queue or a faster consumer is needed.
3. Each sample takes 5 ms of work plus 100 ms of delay, so the period is 105 ms. Over 1,000 samples, the drift is 5 ms × 1,000 = 5 seconds. The `vTaskDelayUntil` call keeps the period at 100 ms and prevents the drift.
4. Each increment is a load, an add and a store, and two cores can interleave those steps, so one increment can be lost. The two fixes are a mutex around the increment, and a design where each variable has one owner, with data passed by queue.
5. The problem is priority inversion. The FreeRTOS feature that fixes it is priority inheritance, which is used by mutexes created with `xSemaphoreCreateMutex`.
6. The problem is deadlock. The rule that prevents it is a fixed lock order, where every task takes lock X before lock Y. A timeout on each take detects a deadlock, but it does not prevent it.

---

## Sources

*Read:*

- The Wikipedia article titled **Real-time operating system**, for the concepts of scheduling and determinism
- The Wikipedia article titled **Priority inversion**, for the problem and the solutions
- The Wikipedia article titled **Deadlock**, for the general definition
- The Wikipedia article titled **Race condition**, for the shared-data problem
- The Wikipedia article titled **Mars Pathfinder**, which describes the priority inversion anomaly and the fix
- The FreeRTOS documentation at freertos.org, for the API reference, the scheduling model, the queue and the mutex. Read the sections on tasks, queues, semaphores and the tick
- The ESP-IDF Programming Guide for FreeRTOS on ESP32 at docs.espressif.com. Read the sections on the FreeRTOS configuration and the tick rate
- The PlatformIO documentation at docs.platformio.org, for the `platformio.ini` format and the ESP32 platform

*Tutorials:*

- The official Arduino-ESP32 examples for FreeRTOS, found in the Arduino-ESP32 core documentation on GitHub under the examples folder. Search for "Arduino-ESP32 FreeRTOS examples"
- The GeeksforGeeks article on real-time operating systems. Search the GeeksforGeeks site for "real time operating system"

*Watch:*

- MIT OpenCourseWare course 6.004, Computation Structures, for the hardware basis of concurrency and memory access. Search for the course title on the MIT OpenCourseWare site
- The Espressif YouTube channel, which has videos on ESP32 and FreeRTOS. Search for "Espressif ESP32 FreeRTOS tutorial"

*Practice:*

- Wokwi at wokwi.com, which supports the ESP32 and can run FreeRTOS-based projects in the browser. Build the pipeline there first. The free tier is enough for this lesson, but its timing does not match real hardware, so repeat the race demo on the bench.

---

## Tomorrow

Day 25 is the module project. You will combine the sensors, the buses, the state machine, the telemetry and the tasks from this module into one embedded device. The device will read several sensors, drive an actuator, send telemetry over a serial link and handle its faults in a structured way. The RTOS task design from today and the layered architecture from Day 23 are both used there, so the choices you made in those lessons will decide how cleanly the final project fits together.
