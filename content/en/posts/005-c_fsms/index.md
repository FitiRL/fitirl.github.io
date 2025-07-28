---
title: "State machines in embedded C"
date: 2025-07-28
author: Fiti
description: FSMs in C as a pattern for embedded systems design.
tags:
  - c
  - pattern
  - fsm
---
## Introduction and motivation

In the world of embedded systems, sometimes a deterministic, predictable, easy-to-debug behavior modeling is desirable when the software is being implemented. This can be achieved in many ways, but one of them is by implementing an FSM (a Finite State Machine).

{{< mermaid >}}
stateDiagram-v2
    STARTUP --> RED : Initialization Complete
    RED --> GREEN : Timer Expired (6s)
    GREEN --> YELLOW : Pedestrian Button + Timer > 20s
    YELLOW --> RED : Timer Expired (2s)
{{< /mermaid >}}

In embedded systems, it is very common to implement reactive systems, and one way to work is with FSM. Think about an elevator or a semaphore that is constantly following simple and static rules to execute one instruction or another: execute this, wait these seconds, during this wait I want to allow this thing, turn this LED on...

## FSM in the software

This scheme/framework/pattern means a different way to program software. On one side, we define the algorithm that the software will run: it will execute what we call "states", and the rules to execute these states. On the other side, completely isolated, we implement the states themselves. Having this decoupled gives us the advantage of adding states and modifying them without affecting the rest of the software flow.

This approach has the main drawback of efficiency, as it consumes more memory and processing time (implementing functors and constantly calling functions) than other alternatives. However, it is very, very clear, easy to debug, scales quite well, and allows you to easily modularize the software. It also allows you to abstract the logic of the software from the states, creating a relation between the business logic and the execution flow of the software.

This scheme is not designed for multithreading environments. One state goes after the previous one.

However, using FSMs, we gain some important advantages:
- Logging states isolates the area of execution. If we know that the FSM is in a specific state, we know that only one function can be executed, which eases debugging. Example: The software has a bug that, for some reason, does not jump from one state to another → With the FSM design approach, the debugging task will be reduced to a few lines without even turning on the debugger!
- We can represent and print the software as an FSM diagram, so we can present it to non-technical users (QA, clients...), which eases the design stage of the software's lifecycle. In some cases, these states do not mean something useful to those users, but we can maintain coherence between this diagram and another, more user-friendly one. This will be much easier than maintaining coherence between the requirements document and the software itself, especially for complex software.
- We reduce the impact of changes. If our software is very, very big, by adding a few lines we can ensure only a new state is added, and the risk of errors is considerably reduced. Scalability is well accepted here.

### Real example: Design a simple printer

In order to explain this idea better, we can design an example, where we will model a firmware FSM for a printer.

> Note: This code is just an example. Optimization, error checking, and modularization can be applied, but it has been simplified to make things simple.

 
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define SECONDS_TO_PRINT 5

// Logic for FSM
enum event_type {
    EVENT_PRINT_REQUEST = 0,
    EVENT_ERROR_NO_PAPER,
    EVENT_CANCEL_COMMAND,
    EVENT_CONTINUE_PRINTING,
    EVENT_CLEAR_ERROR,
    _N_EVENT_MAX
};

enum state_id {
    STATE_READY,
    STATE_PRINTING,
    STATE_ERROR,
    _N_STATE_MAX
};

struct fsm_event {
    enum event_type type;
    void* data;
};

struct fsm; // Forward declaration.

struct fsm_state {
    enum state_id id;
    const char* name;
    void (*on_entry)(struct fsm* fsm);
    void (*on_exit)(struct fsm* fsm);
    enum state_id (*handler)(struct fsm* fsm, struct fsm_event* event);
};

struct fsm {
    struct fsm_state* states[_N_STATE_MAX];
    enum state_id current_state;
    enum state_id previous_state;

    int seconds_printing;
};

void fsm_destroy(struct fsm* fsm) {
    if (fsm) {
        free(fsm);
    }
}

int fsm_transition(struct fsm* fsm, enum state_id new_state) {
    if (new_state >= _N_STATE_MAX) {
        return -1;
    }

    struct fsm_state* current = fsm->states[fsm->current_state];
    struct fsm_state* next = fsm->states[new_state];

    printf("\n##### TRANSITION: %s -> %s\n", current->name, next->name);

    if (current->on_exit) {
        current->on_exit(fsm);
    }

    fsm->previous_state = fsm->current_state;
    fsm->current_state = new_state;

    if (next->on_entry) {
        next->on_entry(fsm);
    }

    return 0;
}

void fsm_handle_event(struct fsm* fsm, struct fsm_event* event) {
    struct fsm_state* current_state = fsm->states[fsm->current_state];

    printf("\nHandling event: %d in state: %s\n", event->type, current_state->name);

    enum state_id next_state = current_state->handler(fsm, event);

    if (next_state != fsm->current_state) {
        fsm_transition(fsm, next_state);
    }
}

void fsm_print_status(struct fsm* fsm) {
    struct fsm_state* current = fsm->states[fsm->current_state];
    printf("\n=== FSM Status ===\n");
    printf("Current State: %s\n", current->name);
    printf("==================\n");
}

// States implementation

void ready_entry(struct fsm* fsm) {
    printf("Entering READY state...\n");
}
void ready_exit(struct fsm* fsm) {
    printf("Exiting READY state ... \n");
}

enum state_id ready_handler(struct fsm* fsm, struct fsm_event* event) {
    switch (event->type) {
        // Someone requested a new print.
        case EVENT_PRINT_REQUEST:
            return STATE_PRINTING;
        default:
            printf("Wait for some job..\n");
            return STATE_READY;
    }
}

void printing_entry(struct fsm* fsm) {
    printf("Starting to print...\n");
    fsm->seconds_printing = 0;
}

void printing_exit(struct fsm* fsm) {
    printf("Finished printing!\n");
}

enum state_id printing_handler(struct fsm* fsm, struct fsm_event* event) {
    fsm->seconds_printing++;
    switch (event->type) {
        case EVENT_ERROR_NO_PAPER:
            printf("Error detected! No paper present\n");
            return STATE_ERROR;
        case EVENT_CANCEL_COMMAND:
            printf("Someone cancelled the printing!\n");
            return STATE_READY;
        case EVENT_CONTINUE_PRINTING:
        default:
            if (fsm->seconds_printing >= SECONDS_TO_PRINT) {
                // The print finished.
                return STATE_READY;
            } else {
                printf("Printing...(%d/%d)\n",
                        fsm->seconds_printing,
                        SECONDS_TO_PRINT);
                return STATE_PRINTING;
            }
    }
}

void error_entry(struct fsm* fsm) {
    printf("An error has been detected!\n");
}

void error_exit(struct fsm* fsm) {
    printf("Exiting error state\n");
}
enum state_id error_handler(struct fsm* fsm, struct fsm_event* event) {
    switch (event->type) {
        case EVENT_CLEAR_ERROR:
            return STATE_READY;
        default:
            printf("In error status, please remove it by launching CLEAR_ERROR\n");
            return STATE_ERROR;
    }
}

struct fsm_state ready_state = {
    .id = STATE_READY,
    .name = "READY",
    .on_entry = ready_entry,
    .on_exit = ready_exit,
    .handler = ready_handler
};

struct fsm_state printing_state = {
    .id = STATE_PRINTING,
    .name = "PRINTING",
    .on_entry = printing_entry,
    .on_exit = printing_exit,
    .handler = printing_handler
};

struct fsm_state error_state = {
    .id = STATE_ERROR,
    .name = "ERROR",
    .on_entry = error_entry,
    .on_exit = error_exit,
    .handler = error_handler
};


struct fsm* fsm_create(enum state_id initial_state) {
    struct fsm* fsm = malloc(sizeof(struct fsm));
    if (!fsm) return NULL;

    memset(fsm, 0, sizeof(memset));

    fsm->states[STATE_PRINTING] = &printing_state;
    fsm->states[STATE_READY] = &ready_state;
    fsm->states[STATE_ERROR] = &error_state;

    fsm->current_state = initial_state;
    fsm->previous_state = initial_state;

    fsm->seconds_printing = 0;

    if (fsm->states[initial_state]->on_entry) {
        fsm->states[initial_state]->on_entry(fsm);
    }

    return fsm;
}


void simulate_printer() {
    printf("Starting FSM...");

    struct fsm* printer_fsm = fsm_create(STATE_READY); // Start as READY
    if (!printer_fsm) {
        printf("Failed to create FSM\n");
        return;
    }

    struct fsm_event events[] = {
        {EVENT_PRINT_REQUEST, NULL},
        {EVENT_CANCEL_COMMAND, NULL},
        {EVENT_PRINT_REQUEST, NULL},
        {EVENT_CONTINUE_PRINTING, NULL},
        {EVENT_CONTINUE_PRINTING, NULL},
        {EVENT_CONTINUE_PRINTING, NULL},
        {EVENT_CONTINUE_PRINTING, NULL},
        {EVENT_CONTINUE_PRINTING, NULL},
        {EVENT_PRINT_REQUEST, NULL},
        {EVENT_ERROR_NO_PAPER, NULL},
        {EVENT_PRINT_REQUEST, NULL},
        {EVENT_CLEAR_ERROR, NULL},
    };

    int num_events = sizeof(events) / sizeof(events[0]);

    for (int i = 0; i < num_events; i++) {
        printf("\n--- Simulation Step %d ---\n", i + 1);
        fsm_print_status(printer_fsm);

        fsm_handle_event(printer_fsm, &events[i]);

        printf("Press Enter to continue...");
        getchar();
    }

    fsm_print_status(printer_fsm);
    fsm_destroy(printer_fsm);
    printf("#####################\n");
    printf("Simulation completed!\n");
}

int main() {
    simulate_printer();
    return 0;
}
```

{{< mermaid >}}
stateDiagram-v2
    READY --> PRINTING : EVENT_PRINT_REQUEST
    READY --> READY : Other events (wait for job)
    
    PRINTING --> READY : EVENT_CANCEL_COMMAND
    PRINTING --> READY : Print completed (5 seconds)
    PRINTING --> ERROR : EVENT_ERROR_NO_PAPER
    PRINTING --> PRINTING : EVENT_CONTINUE_PRINTING (not finished)
    
    ERROR --> READY : EVENT_CLEAR_ERROR
    ERROR --> ERROR : Other events (stay in error)
{{< /mermaid >}}

Maintaining the firmware is much simpler now. Take these two examples:
- To handle a new printing error, just add a new state and tweak the printing state handler to transition into it. The change is small, localized, and unlikely to introduce mistakes.
- If the system gets stuck in an error state, there's only one function to inspect, making debugging faster and more focused.


## Conclusions

State machines are a powerful pattern of programming, especially in embedded systems. They allow writing deterministic code execution paths and separating the logic of execution from the logic of the software itself. They help organize the code better, easing maintainability and debugging. However, this pattern is not suitable in some environments, like multithreading ones or when the task of the software cannot be modeled easily as an FSM.
