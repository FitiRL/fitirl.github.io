---
title: "Readable, Searchable, Actionable - The Power of Structured Logs"
date: 2025-07-16
author: Fiti
description: Structured log example in C
tags:
  - c
  - logging
---

## Introduction

Logs are very important in any system. They allow us to know what is happening, leaving a trace in the system that we can analyze, for debugging or for performing a forensic analysis of the system. In embedded devices, this tool gains a lot of importance because sometimes (especially in an IoT environment), we have thousands of devices running, and it is hard to detect a failure in one of them. That's where logs and monitoring (which is not being covered in this article) come into play.

## Normal logs (unstructured) vs structured logs

We will agree that logs should contain some **necessary information**. For example, a timestamp is strictly required. The source is also very important. But the message is up to the user (the developer) to define, and then things become freer (and with freedom, inconsistency). With inconsistency comes mess.

That's the problem that structured logs try to solve. A log message like:

```shell
2025-06-28T00:47:15Z: [ERROR] Failed to connect to database: timeout after 5000ms
```

Would be converted to something like:

```json
{
  "timestamp": "2025-06-28T00:47:15Z",
  "level": "error",
  "message": "Failed to connect to database",
  "module": "database",
  "error_code": "TIMEOUT",
  "duration_ms": 5000
}
```

### Advantages vs disadvantages

If structured logs were better than unstructured ones, they would always be used instead. However, structured logs have drawbacks and should only be used in specific situations.

#### Advantages

- Structured logs can be loaded automatically in some log aggregators.
- Analysis of these logs can be automated.
- Messages can be filtered by any field or by any specific error code, without any transformation.
- As all messages are generated in the same way, they are consistent. By reducing the developer's freedom, all log messages follow the same format, and human error is considerably reduced.

#### Disadvantages
- The size of the logs is larger.
- Readability: Structured formats are harder for a human to read: easy for the machine, hard for the human.

### When structured logs should be used?

Structured logs are suitable for some specific scenarios:

- For unifying the log schema between devices.
- For analysis of logs, where they are treated like metrics and used, for example, to obtain statistics.

> Hybrid: A valid approach (as the example will show) would be to use non-structured logs during debugging or to make it configurable to use structured or unstructured logs depending on the requirements at the moment.

## Example of structured logs implementation in C

But implementing structured logs is not that hard! Here we will **develop our custom log library**, very, very simple (but better than using `printf()`). And most important: <u>without any third-party dependency</u>.

1. First of all, define the structures we are going to use: the `log_level_t` structure with different log levels, its string representation, and the logger structure.
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <stdarg.h>

#define TIMESTAMP_MAX_LEN 32
#define MESSAGE_MAX_LEN 1024
#define LOGLINE_MAX_LEN 1024 + 512

typedef enum {
    LOG_DEBUG = 0,
    LOG_INFO,
    LOG_WARN,
    LOG_ERROR,
    LOG_FATAL,
    _N_LOG_LEVEL
} log_level_t;
// String representation
static const char* level_names[] = {"DEBUG", "INFO", "WARN", "ERROR", "FATAL"};

typedef struct {
    FILE* file;
    int log_to_console;
    log_level_t min_level;
} logger_t;

static logger_t g_logger = {NULL, 1, LOG_INFO};

```
3. A helper function that gets the timestamp:
```c
static void _get_timestamp(char* buffer, size_t size) {
    time_t now = time(NULL);
    struct tm* tm_info = localtime(&now);
    strftime(buffer, size, "%Y-%m-%d %H:%M:%S", tm_info);
    buffer[size - 1] = '\0';
}
```
4. The public `init()` function:
```c
int logger_init(const char* filename, int console, log_level_t min_level) {
    g_logger.log_to_console = console;
    g_logger.min_level = min_level;
    
    if (filename) {
        g_logger.file = fopen(filename, "a");
        if (!g_logger.file) {
            fprintf(stderr, "Failed to open log file: %s\n", filename);
            return -1;
        }
    }
    return 0;
}
```
5. Of course, the `destroy()` function:
```c
void logger_cleanup() {
    if (g_logger.file) {
        fclose(g_logger.file);
        g_logger.file = NULL;
    }
}
```
6. And finally, the most important function: the logger one:
```c
void logger(log_level_t level, const char* component, const char* event, ...) {
    if (level < g_logger.min_level) {
        return;
    }
    
    char timestamp[TIMESTAMP_MAX_LEN];
    _get_timestamp(timestamp, sizeof(timestamp));
    
    char log_line[LOGLINE_MAX_LEN];
    int offset = 0;
    
    offset += snprintf(log_line + offset, sizeof(log_line) - offset,
                      "{\"timestamp\":\"%s\",\"loglevel\":\"%s\",\"component\":\"%s\",\"event\":\"%s\"",
                      timestamp, level_names[level], 
                      component ? component : "MAIN", event);
    
    va_list args;
    va_start(args, event);
    
    const char* key = NULL;
    const char* value = NULL;
    while ((key = va_arg(args, const char*)) != NULL) {
        value = va_arg(args, const char*);
        if (value) {
            offset += snprintf(log_line + offset, sizeof(log_line) - offset,
                             ",\"%s\":\"%s\"", key, value);
        }
    }
    va_end(args);
    
    offset += snprintf(log_line + offset, sizeof(log_line) - offset, "}\n");
    
    if (g_logger.log_to_console) {
        printf("%s", log_line);
        fflush(stdout);
    }
    
    if (g_logger.file) {
        fprintf(g_logger.file, "%s", log_line);
        fflush(g_logger.file);
    }
}
```

We can then change our lines:

```c
printf("Warning: There is not left space in external module\n");
```

by

```c
logger(LOG_WARN, "MAIN", "no_left_space",
        "trigger", "storage_module",
        "location", "external_module",
        "job_id", "1224",
        NULL);
```

And the output would be sighly different:

```plain
Warning: There is not left space in external module
```

vs 

```json
{"timestamp":"2025-06-28 02:19:58","loglevel":"WARN","component":"MAIN","event":"no_left_space","trigger":"storage_module","location":"external_module","job_id":"1224"}
```

### Extra: Hybrid logging

As a good practice, it is recommended to enable debug mode via CFLAG, with something like:

```c
#ifdef _DEBUG
    if (logger_init("example.log", 1, LOG_DEBUG) != 0) {
        return 1;
    }
#else
    if (logger_init("example.log", 0, LOG_INFO) != 0) {
        return 1;
    }
#endif
```

We can even add to `logger_init()` the format (json vs plain), to make it more readable during the early development or debugging stages.
## Full code

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <stdarg.h>

#define TIMESTAMP_MAX_LEN 32
#define MESSAGE_MAX_LEN 1024
#define LOGLINE_MAX_LEN 1024 + 512

typedef enum {
    LOG_DEBUG = 0,
    LOG_INFO,
    LOG_WARN,
    LOG_ERROR,
    LOG_FATAL,
    _N_LOG_LEVEL
} log_level_t;
// String representation
static const char* level_names[] = {"DEBUG", "INFO", "WARN", "ERROR", "FATAL"};

typedef struct {
    FILE* file;
    int log_to_console;
    log_level_t min_level;
} logger_t;

static logger_t g_logger = {NULL, 1, LOG_INFO};

static void _get_timestamp(char* buffer, size_t size) {
    time_t now = time(NULL);
    struct tm* tm_info = localtime(&now);
    strftime(buffer, size, "%Y-%m-%d %H:%M:%S", tm_info);
    buffer[size - 1] = '\0';
}

int logger_init(const char* filename, int console, log_level_t min_level) {
    g_logger.log_to_console = console;
    g_logger.min_level = min_level;
    
    if (filename) {
        g_logger.file = fopen(filename, "a");
        if (!g_logger.file) {
            fprintf(stderr, "Failed to open log file: %s\n", filename);
            return -1;
        }
    }
    return 0;
}

void logger(log_level_t level, const char* component, const char* event, ...) {
    if (level < g_logger.min_level) {
        return;
    }
    
    char timestamp[TIMESTAMP_MAX_LEN];
    _get_timestamp(timestamp, sizeof(timestamp));
    
    char log_line[LOGLINE_MAX_LEN];
    int offset = 0;
    
    offset += snprintf(log_line + offset, sizeof(log_line) - offset,
                      "{\"timestamp\":\"%s\",\"loglevel\":\"%s\",\"component\":\"%s\",\"event\":\"%s\"",
                      timestamp, level_names[level], 
                      component ? component : "MAIN", event);
    
    va_list args;
    va_start(args, event);
    
    const char* key = NULL;
    const char* value = NULL;
    while ((key = va_arg(args, const char*)) != NULL) {
        value = va_arg(args, const char*);
        if (value) {
            offset += snprintf(log_line + offset, sizeof(log_line) - offset,
                             ",\"%s\":\"%s\"", key, value);
        }
    }
    va_end(args);
    
    offset += snprintf(log_line + offset, sizeof(log_line) - offset, "}\n");
    
    if (g_logger.log_to_console) {
        printf("%s", log_line);
        fflush(stdout);
    }
    
    if (g_logger.file) {
        fprintf(g_logger.file, "%s", log_line);
        fflush(g_logger.file);
    }
}

void logger_cleanup() {
    if (g_logger.file) {
        fclose(g_logger.file);
        g_logger.file = NULL;
    }
}

int main() {
#ifdef _DEBUG
    if (logger_init("example.log", 1, LOG_DEBUG) != 0) {
        return 1;
    }
#else
    if (logger_init("example.log", 1, LOG_INFO) != 0) {
        return 1;
    }
#endif

    printf("Warning: There is not left space in external module\n");
    
    logger(LOG_WARN, "MAIN", "no_left_space",
        "trigger", "storage_module",
        "location", "external_module",
        "job_id", "1224",
        NULL);
    
    logger_cleanup();
    
    printf("\nCheck 'example.log' file for the logged messages!\n");
    return 0;
}
```

## Conclusions

Structured logs are helpful in certain scenarios. Analyzing logs automatically (like loading them into a database) can be really tough if the format isn't structured. But with structure comes a trade-off: logs get bigger, may impact efficiency, and are harder for humans to read. 

On the flip side, structured logging cuts down human error and makes everything more uniform (we all know how different messages can be depending on the developer).
When designing a logging system, understanding these trade-offs is key. Structured logs shine in environments where scalability, automation, and consistency are priorities such as IoT systems, cloud infrastructures, or enterprise-grade applications. They allow machines to parse, filter, and correlate events quickly across devices and services, reducing the need for manual inspection.

However, developers working on smaller projects or those focused on rapid prototyping might favor unstructured logs for their simplicity and readability. The decision should ultimately balance technical needs with human usability. Choose structure when precision and automation are necessary, and opt for plain logs when speed and straightforward comprehension matter more.
A hybrid approach can bring flexibility. Use structure when it's most beneficial, and fall back to simplicity when clarity and agility are more important.
