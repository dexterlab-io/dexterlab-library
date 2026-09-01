# WAIT / POLL / DUMP Mechanisms — DexterLab Parser (Version 1.0)
WAIT, POLL, and DUMP are the most advanced multi‑cycle and multi‑beat mechanisms
implemented in the DexterLab Command Parser.  
They provide timing control, conditional execution, memory inspection, and file
output capabilities.

This document describes:
- WAIT timing mechanism
- POLL retry/timeout/mask/toggle logic
- DUMP memory and file operations
- AXI interaction
- FSM behavior for each mechanism

---

# 📌 WAIT Mechanism

WAIT introduces a controlled delay in the execution flow.

### Example

WAIT 10 ms
WAIT 5 us
WAIT 100 cycles

Codice

### Fields used
- `wait_value`
- `wait_unit`

### Internal signals
- `s_wait_active`
- `s_wait_done`
- `s_wait_count`

### Behavior
1. FSM enters `exec_wait`.
2. WAIT counter starts (`s_wait_active = '1'`).
3. Logger prints WAIT **only once**, at the first cycle.
4. Counter increments until:
s_wait_count = wait_value

Codice
5. FSM transitions to `next_cmd`.

WAIT is **purely internal** and does not interact with AXI.

---

# 📌 POLL Mechanism

POLL repeatedly reads a memory location until a condition is met or a timeout occurs.

### Supported commands
- `POLL_READ`
- `POLL_TOGGLE`
- `POLL_PING`

### Example
POLL_READ 0x4000 DATA=0x01 MASK=0xFF TIMEOUT=100ms
POLL_TOGGLE 0x4000 MASK=0x01 TIMEOUT=50ms
POLL_PING 0x4000 TIMEOUT=200ms

Codice

### Fields used
- `addr`
- `data`
- `data_array(0)` → mask
- `timeout`
- `wait_value` → retry delay

### Internal signals
- `s_poll_retry_count`
- `s_poll_delay_count`
- `s_poll_timeout_expired`
- `s_ack_data`
- `s_ack_err`

---

## 📌 POLL_READ Logic

Condition:
(rsp_data AND mask) = (expected AND mask)

Codice

If true → POLL ends successfully.  
If false → retry.

---

## 📌 POLL_TOGGLE Logic

Condition:
(rsp_data AND mask) toggles compared to previous value

Codice

Used for detecting edges or activity.

---

## 📌 POLL_PING Logic

Condition:
rsp_valid = '1'

Codice

Used to detect device responsiveness.

---

## 📌 Retry and Timeout

### Retry
After each failed attempt:
- FSM enters `exec_poll_wait`
- waits `wait_value` cycles
- returns to `exec_poll`

### Timeout
If:
retry_count * wait_value >= timeout

Codice
→ POLL fails.

Timeout is logged and counted in:
- `s_poll_error_count`

---

# 📌 DUMP Mechanism

DUMP reads memory and logs each beat.  
It supports both console logging and file output.

### Supported commands
- `DUMP_LEN`
- `DUMP_RANGE`
- `DUMP_FILE`
- `DUMP_FILE_CHECK`

### Examples
DUMP_LEN 0x5000 LEN=32
DUMP_RANGE 0x5000 0x5040
DUMP_FILE 0x5000 LEN=32 "dump.bin"
DUMP_FILE_CHECK 0x5000 LEN=32 "expected.bin"

Codice

### Fields used
- `addr`
- `len`
- `filename`

---

## 📌 DUMP_LEN

Reads `len` beats starting at `addr`.

---

## 📌 DUMP_RANGE

Reads from:
addr_start → addr_end

Codice

---

## 📌 DUMP_FILE

Writes each beat to a file:
- automatic file open
- beat‑by‑beat write
- automatic file close

---

## 📌 DUMP_FILE_CHECK

Compares memory content with a reference file:
- logs mismatches
- increments error counters
- prints summary

---

# 📌 AXI Interaction

WAIT → no AXI  
POLL → AXI READ on each retry  
DUMP → AXI READ for each beat

### Signals used
- `s_cmd_valid`
- `cmd_ready`
- `rsp_in.rsp_valid`
- `rsp_in.rsp_data`
- `rsp_in.rsp_last`
- `s_ack_data`
- `s_ack_err`

---

# 📌 FSM Behavior

### WAIT
exec_wait → next_cmd

Codice

### POLL
exec_poll → exec_poll_wait → exec_poll → next_cmd

Codice

### DUMP
exec_dump → next_cmd

Codice

All three mechanisms rely on the micro‑command pipeline to ensure correct parameters.

---

# 📌 Version Notes
This document describes WAIT/POLL/DUMP mechanisms for parser version **1.0**.

Version 1.1 may:
- unify POLL and WAIT timing logic  
- introduce extended DUMP formats  
- simplify retry/timeout handling  
- add binary dump modes  
