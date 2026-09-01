# DexterLab Parser FSM — Version 1.0
The DexterLab Command Parser uses a structured finite-state machine (FSM) to control
the execution flow of micro-commands. The FSM ensures deterministic behavior, clean
AXI handshakes, and correct sequencing of multi-beat and multi-cycle operations.

This document describes:
- the full list of FSM states
- the purpose of each state
- the transition rules
- the AXI interaction model
- the relationship between FSM and micro-command pipeline

---

## 📌 FSM Overview
The FSM is composed of **15 states**, grouped into three categories:

### 1. Parsing states
- `idle`
- `parse_file`
- `parse_file_done`

### 2. Command sequencing states
- `next_cmd`

### 3. Execution states
- `exec_print`
- `exec_atomic`
- `exec_wait`
- `exec_base`
- `exec_fifo`
- `exec_burst`
- `exec_wrap`
- `exec_fill`
- `exec_poll`
- `exec_poll_wait`
- `exec_dump`
- `finished`

Each execution state corresponds to a specific micro-command type.

---

## 📌 State Descriptions

### **idle**
Initial state.  
Waits for `start = '1'`.  
Resets internal counters and opens the log file.

---

### **parse_file**
Reads one line from the command file.  
Performs:
- lexical analysis  
- syntactic parsing  
- semantic validation  
- micro-command generation  

Handles include stack and EOF.

---

### **parse_file_done**
Triggered when a full line has been parsed.  
Loads the first two micro-commands into the pipeline:

- `s_mc_cur`
- `s_mc_next`

If no micro-commands exist, transitions to `finished`.

Otherwise transitions to `next_cmd`.

---

### **next_cmd**
Advances the micro-command pipeline:

- `s_mc_cur <= s_mc_next`
- `s_mc_next <= next micro-command`

Then selects the correct execution state based on:

- `s_mc_cur.cmd_type`

This is the **central dispatch state**.

---

## 📌 Execution States

### **exec_print**
Prints a text string to the log.

---

### **exec_atomic**
Sets or clears the atomic flag.  
Used to enforce exclusive access to AXI operations.

---

### **exec_wait**
Implements multi-cycle waiting.  
Uses:
- `s_wait_active`
- `s_wait_done`
- `s_wait_count`

Transitions back to `next_cmd` when the wait completes.

---

### **exec_base**
Handles single-beat AXI commands:
- READ
- WRITE
- CHECK

Uses centralized AXI acknowledgments:
- `s_ack_data`
- `s_ack_err`

Transitions to `next_cmd` after the beat completes.

---

### **exec_fifo**
Handles multi-beat FIFO operations:
- FIFO_WRITE
- FIFO_READ
- FIFO_CHECK

Uses:
- `s_data_index`
- `s_ack_data`
- `s_ack_err`

Transitions to `next_cmd` when `rsp_last = '1'`.

---

### **exec_burst**
Handles AXI burst operations:
- BURST_WRITE
- BURST_READ
- BURST_CHECK

Identical structure to FIFO, but with burst semantics.

---

### **exec_wrap**
Handles WRAP operations (AXI wrapping bursts).

---

### **exec_fill**
Implements memory fill operations:
- FILL_LEN
- FILL_RANGE
- FILL_CHECK

---

### **exec_poll**
Implements POLL logic:
- masked comparisons
- retry logic
- timeout logic
- toggle detection
- ping behavior

Transitions to:
- `exec_poll_wait` (if delay required)
- `next_cmd` (if done or timeout)

---

### **exec_poll_wait**
Implements delay between POLL retries.  
Uses `s_poll_delay_count`.

---

### **exec_dump**
Implements memory dump operations:
- DUMP_LEN
- DUMP_RANGE
- DUMP_FILE
- DUMP_FILE_CHECK

Handles file open/close and beat-by-beat logging.

---

### **finished**
Final state.  
Prints summary and closes log file.  
Transitions back to `idle`.

---

## 📌 FSM Diagram (Mermaid)

```mermaid
stateDiagram-v2
    idle --> parse_file
    parse_file --> parse_file_done

    parse_file_done --> next_cmd
    parse_file_done --> finished

    next_cmd --> exec_print
    next_cmd --> exec_atomic
    next_cmd --> exec_wait
    next_cmd --> exec_base
    next_cmd --> exec_fifo
    next_cmd --> exec_burst
    next_cmd --> exec_wrap
    next_cmd --> exec_fill
    next_cmd --> exec_poll
    next_cmd --> exec_dump
    next_cmd --> parse_file

    exec_wait --> next_cmd
    exec_base --> next_cmd
    exec_fifo --> next_cmd
    exec_burst --> next_cmd
    exec_wrap --> next_cmd
    exec_fill --> next_cmd
    exec_poll --> next_cmd
    exec_dump --> next_cmd

    exec_poll_wait --> exec_poll

    finished --> idle

📌 Relationship with Micro-command Pipeline
The FSM relies on the pipeline to ensure:

s_mc_cur is valid when entering any exec_* state

AXI commands are generated with correct parameters

logging prints correct values

WAIT/POLL/DUMP operate on correct micro-command fields

Without the pipeline, the FSM would enter execution states with stale data.

📌 Version Notes
This documentation describes FSM version 1.0, which uses the two-stage micro-command pipeline.

Version 1.1 (optional future redesign) may:

remove the pipeline

simplify state transitions

unify multi-beat execution logic

But this requires a full architectural rewrite.
