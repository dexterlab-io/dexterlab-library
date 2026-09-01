# DexterLab Command Parser — Version 1.0
The DexterLab Command Parser is a sequential interpreter that reads command files
(`cmd.txt` and included files), parses them, converts them into micro-commands, and
executes them through an AXI-like interface connected to the arbiter/wrapper.

This document describes:
- overall architecture
- micro-command pipeline (`s_mc_cur` / `s_mc_next`)
- FSM flow
- AXI interface behavior
- WAIT / POLL / DUMP mechanisms
- logging system

---

## 📌 Overview
The parser is composed of five main blocks:

1. **PROCESS_FILE**  
   Reads one line from the command file, performs lexical, syntactic, and semantic
   parsing, generates a list of micro-commands (`s_mc_list`), and updates the include
   stack, EOF status, and parsing errors.

2. **FSM (synchronous + combinational)**  
   Controls the execution flow, selects which micro-command to run, determines when
   to advance to the next one, and manages all `exec_*` states.

3. **Micro-command Pipeline**  
   - `s_mc_cur` = current micro-command  
   - `s_mc_next` = next micro-command  
   This pipeline ensures that the FSM always has a valid micro-command when changing
   states.

4. **AXI-like Interface**  
   Generates `cmd_valid`, drives `cmd_out`, and processes AXI acknowledgments
   (`s_ack_data`, `s_ack_err`, `s_ack_end`).

5. **Logging System**  
   Writes detailed execution logs to `cmd.log`, including command execution,
   multi-beat operations, AXI errors, execution errors, and a final summary.

---

## 📌 Micro-command Pipeline (critical)
The parser uses a **two-stage pipeline**:

- Stage 1: `s_mc_cur`  
- Stage 2: `s_mc_next`

This pipeline is **required** because:

- the FSM changes state *before* the synchronous process updates registers
- combinational processes (AXI, logging, WAIT, POLL, DUMP) read `s_mc_cur`
- without the pipeline, the FSM would enter `exec_*` states with stale or empty
  micro-command data

### Pipeline flow
1. In `parse_file_done`:
   - `s_mc_cur <= s_mc_list(0)`
   - `s_mc_next <= s_mc_list(1)`

2. In `next_cmd`:
   - `s_mc_cur <= s_mc_next`
   - `s_mc_next <= s_mc_list(s_mc_index + 2)`

This guarantees that **every `exec_*` state receives the correct micro-command**.

---

## 📌 FSM — State Machine Structure
The FSM includes the following states:

- `idle`
- `parse_file`
- `parse_file_done`
- `next_cmd`
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

### FSM Diagram (Mermaid)
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

    finished --> idle

📌 AXI-like Interface
The parser interacts with AXI_FULL using:

s_cmd_valid

s_cmd_out

cmd_ready

rsp_in

Key rule
cmd_valid is asserted only when the FSM enters an AXI-related exec_* state.

This ensures:

one command per state entry

no duplicate requests

clean handshake with AXI_FULL

📌 WAIT Mechanism
WAIT is a multi-cycle operation controlled by:

s_wait_active

s_wait_done

s_wait_count

The logger prints the WAIT command only on the first cycle, when the WAIT starts.

📌 POLL Mechanism
POLL supports:

masked comparisons

retry logic

timeout logic

toggle detection

delay between retries

Supported commands:

POLL_READ

POLL_TOGGLE

POLL_PING

The parser uses centralized AXI acknowledgments (s_ack_data, s_ack_err) to simplify POLL behavior and ensure consistent retry/timeout handling.

📌 DUMP Mechanism
DUMP supports:

normal memory dump

dump to file

dump file check

Features:

automatic file open/close

beat-by-beat logging

address and data verification

📌 Logging System
The logger writes:

every executed command

every beat of multi-beat operations

AXI errors

execution errors

a final summary

Output file: cmd.log

📌 Final Summary
At the end of execution, the parser prints:

parsing error count

execution error count (per category)

AXI error count

total errors

final result: OK or FAIL

📌 Version 1.0 Notes
This documentation describes version 1.0, which uses the micro-command pipeline.

A future version (1.1) may:

remove the pipeline

simplify the FSM

use single-stage micro-command execution

But this requires a full redesign of the state machine.
