# DexterLab Micro‑Commands — Version 1.0
Micro‑commands (MCs) are the fundamental execution units of the DexterLab Command
Parser. Each parsed command line is converted into one or more micro‑commands,
which are then executed sequentially by the FSM.

This document describes:
- micro‑command structure
- all supported micro‑command types
- parameters and fields
- multi‑beat behavior
- pipeline interaction
- AXI mapping

---

## 📌 Micro‑Command Structure
Each micro‑command is represented by a record:

```vhdl
type t_mc is record
    cmd_type     : t_cmd_type;
    addr         : std_logic_vector(C_ADDR_WIDTH-1 downto 0);
    data         : std_logic_vector(C_DATA_WIDTH-1 downto 0);
    data_array   : t_data_array;
    len          : natural;
    wait_value   : natural;
    wait_unit    : t_time_unit;
    timeout      : natural;
    filename     : t_filename;
    atomic       : std_logic;
end record;

Key fields
cmd_type — identifies the operation

addr — base address

data — single‑beat data

data_array — multi‑beat data (FIFO/BURST/WRAP/FILL)

len — number of beats

wait_value / wait_unit — WAIT/POLL timing

timeout — POLL timeout

filename — DUMP file name

atomic — atomic flag (ON/OFF)

📌 Micro‑Command Pipeline
The parser uses a two‑stage pipeline:

s_mc_cur — micro‑command currently being executed

s_mc_next — next micro‑command to execute

This ensures:

correct timing

correct AXI parameters

correct logging

correct WAIT/POLL behavior

📌 Supported Micro‑Command Types
1️⃣ PRINT
Prints a string to the log.

Codice
PRINT "Hello World"
Fields used:

cmd_type

print_text

2️⃣ ATOMIC
Controls atomic mode.

Codice
ATOMIC ON
ATOMIC OFF
Field:

atomic

3️⃣ WAIT
Implements multi‑cycle waiting.

Codice
WAIT 10 ms
WAIT 5 us
Fields:

wait_value

wait_unit

4️⃣ BASE Commands (single‑beat AXI)
READ
Codice
READ 0x1000
WRITE
Codice
WRITE 0x1000 0xABCD1234
CHECK
Codice
CHECK 0x1000 0xABCD1234
Fields:

addr

data

data_array(0)

5️⃣ FIFO Commands (multi‑beat)
FIFO_WRITE
Codice
FIFO_WRITE 0x2000 [01 02 03 04]
FIFO_READ
Codice
FIFO_READ 0x2000 LEN=4
FIFO_CHECK
Codice
FIFO_CHECK 0x2000 [01 02 03 04]
Fields:

addr

data_array

len

6️⃣ BURST Commands (multi‑beat AXI bursts)
BURST_WRITE
BURST_READ
BURST_CHECK
Identical structure to FIFO, but with AXI burst semantics.

7️⃣ WRAP Commands (AXI wrapping bursts)
WRAP_WRITE
WRAP_READ
WRAP_CHECK
Fields identical to BURST.

8️⃣ FILL Commands
FILL_LEN
Codice
FILL_LEN 0x3000 LEN=16 DATA=0xFF
FILL_RANGE
Codice
FILL_RANGE 0x3000 0x3010 DATA=0xAA
FILL_CHECK
Codice
FILL_CHECK 0x3000 LEN=16 DATA=0xFF
Fields:

addr

len

data_array(0)

9️⃣ POLL Commands
POLL_READ
Codice
POLL_READ 0x4000 DATA=0x01 MASK=0xFF TIMEOUT=100ms
POLL_TOGGLE
Codice
POLL_TOGGLE 0x4000 MASK=0x01 TIMEOUT=50ms
POLL_PING
Codice
POLL_PING 0x4000 TIMEOUT=200ms
Fields:

addr

data

data_array(0) (mask)

wait_value (retry delay)

timeout

🔟 DUMP Commands
DUMP_LEN
Codice
DUMP_LEN 0x5000 LEN=32
DUMP_RANGE
Codice
DUMP_RANGE 0x5000 0x5040
DUMP_FILE
Codice
DUMP_FILE 0x5000 LEN=32 "dump.bin"
DUMP_FILE_CHECK
Codice
DUMP_FILE_CHECK 0x5000 LEN=32 "expected.bin"
Fields:

addr

len

filename

📌 AXI Mapping
Each micro‑command maps to AXI fields:

cmd_out.addr

cmd_out.data

cmd_out.len

cmd_out.cmd_type

cmd_valid

Multi‑beat commands increment:

s_data_index

addr += DATA_WIDTH/8

📌 Version Notes
This document describes micro‑commands for parser version 1.0.

Version 1.1 may:

unify multi‑beat commands

simplify POLL

introduce extended DUMP formats

remove pipeline dependency
