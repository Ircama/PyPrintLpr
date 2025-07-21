# LPR printer client/server with proxy support

RFC 1179 client and server toolkits and Python library for interacting with printers via LPR protocol or RAW mode, as well as a proxy/server for debugging, job capture, and protocol analysis.

## Features

- RFC 1179 LPR protocol implementation
- Easy-to-use Python API for client and server LPR features
- LPR (515) and RAW (9100) protocol support
- Print job simulation and debugging
- Bidirectional TCP/UDP proxy and traffic inspection
- Job queue management and archiving
- Hexdump tracing and protocol analysis
- Cross-platform (Windows, Linux, macOS)

## Installation

```bash
pip install PyPrintLpr
```

## Command Line Usage

```
Usage: python -m pyprintlpr [client|server] [args...]
  client: Run the print client
  server: Run the proxy/server
  args: use --help to get the client and server usage.
```

### Client
Run the client to send print jobs to a printer:

```
usage: pyprintlpr client [-h] -a ADDRESS [-p PORT]
                         (-P | -s LIST | -S LIST | -R LIST | -f PRINT_FILE) [-d]
                         [-e] [-q QUEUE] [-r] [-t TIMEOUT] [-b RECV_BUFFER]
                         [-u USERNAME] [-l LABEL] [-j JOB_NAME]

RAW and LPR print client.

optional arguments:
  -h, --help            show this help message and exit
  -a ADDRESS, --address ADDRESS
                        Printer host name or IP address. (Example: -a 192.168.1.87)
  -p PORT, --port PORT  Printer port (default is "LPR" = 515)
  -P, --print           Send the "Print any waiting jobs" command (RFC1179 0x01) to
                        the specified queue and exit.
  -s LIST, --status LIST
                        Send queue state (short) command with "list" attribute
                        (RFC1179 0x03) to the specified queue and print the result.
  -S LIST, --longstatus LIST
                        Send queue state (long) command with "list" attribute
                        (RFC1179 0x04) to the specified queue and print the result.
  -R LIST, --remove LIST
                        Remove jobs command with "list" attribute (RFC1179 0x05) to
                        the specified queue.
  -f PRINT_FILE, --file PRINT_FILE
                        File to be printed
  -d, --debug           Print debug information
  -e, --epson           Use Epson header and footer
  -q QUEUE, --queue QUEUE
                        Queue name; default queue name is "PASSTHRU"
  -r, --reserved        Use reserved port range 721-731 (RFC 1179 requirement)
  -t TIMEOUT, --timeout TIMEOUT
                        Timeout (default: 5.0 seconds)
  -b RECV_BUFFER, --buffer RECV_BUFFER
                        Receive buffer size (default: 4096 bytes)
  -u USERNAME, --username USERNAME
                        User name (default: current user)
  -l LABEL, --label LABEL
                        Label for the job (optional, used to set the name of source
                        file)
  -j JOB_NAME, --job JOB_NAME
                        Job name (default: "LprClientJob")

RAW/LPR print client.
```

Notes:

- Use the `epson` flag to send Epson-specific headers/footers
- Use the `reserved` flag to comply with RFC 1179 reserved port requirements
- Debugging: Enable logging for protocol analysis

### Server/Proxy

Run the server/proxy to capture, forward, or analyze print jobs:

```
usage: pyprintlpr server [-h] [-a ADDRESS] [-t] [-s] [-p SAVE_PATH] [-q] [-l PORTS]
                         [-e PORTS] [--timeout TIMEOUT]

RAW and LPR print server with proxy, forward and local loopback features.

optional arguments:
  -h, --help            show this help message and exit
  -a ADDRESS, --address ADDRESS
                        Printer IP address (if not provided, runs in local-only
                        mode)
  -t, --trace           Enable hex dump tracing
  -s, --save-files      Save received print jobs to disk
  -p SAVE_PATH, --save-path SAVE_PATH
                        Path name of the directory including the saved jobs
                        (defaut: "lpr_jobs")
  -q, --quiet           Do not print debug data
  -l PORTS, --loopback PORTS
                        Comma-separated list of ports to loopback instead of
                        forwarding to the target IP (e.g., "515,9100")
  -e PORTS, --exclude PORTS
                        Comma-separated list of ports to exclude from tracing
                        (e.g., "515,9100")
  --timeout TIMEOUT     Timeout in seconds for receiving control/data files
                        (default: 10.0 seconds)

RAW/LPR print server.
```

## Python API Usage

### Client API
The client API allows sending print jobs, check queue status, and interact with printers programmatically.

#### Basic Usage

```python
from pyprintlpr import LprClient

with LprClient('192.168.1.100', port="LPR", queue='PASSTHRU') as printer:
    # Send a print job from a file
    with open('document.txt', 'rb') as f:
        printer.send(f.read())
```

#### LprClient Class Reference
- `LprClient(hostname, port=515, queue='PASSTHRU', ...)` – Initialize client

    - **hostname** (`str`):  
      Printer hostname or IP address to connect to.

    - **port** (`int` or `str`, default: `9100`):  
      Port number for the printer. Use `9100` or `RAW` for RAW printing, `515` or `"LPR"` for LPR protocol.

    - **timeout** (`float`, default: `5.0`):  
      Socket timeout in seconds for network operations.

    - **queue** (`str`, default: `"PASSTHRU"`):  
      LPR queue name, used when sending jobs via LPR protocol.

    - **recv_buffer** (`int`, default: `4096`):  
      Size of the receive buffer for socket operations.

    - **username** (`str`, optional):  
      Username for LPR jobs. Defaults to the current system user if not specified.

    - **job_name** (`str`, optional):  
      Job name for LPR jobs. Defaults to `"LprClientJob"` if not specified.

    - **file_name** (`str`, optional):  
      File name for LPR jobs. Defaults to the job name if not specified.

    - **use_reserved_port** (`bool`, default: `False`):  
      If `True`, attempts to use a reserved source port (721-731) as required by RFC 1179 for LPR protocol.

- Methods:

    | **Method**              | **Description**                                                                                                                  |
    | ----------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
    | `connect()`             | Opens a TCP socket connection to the printer at the specified host and port, with timeout.                                       |
    | `disconnect()`          | Gracefully shuts down and closes the socket connection if open.                                                                  |
    | `send(data: bytes)`     | Sends `bytes` directly to the printer over the socket connection.                                                            |
    | `remote_cmd(cmd: str, args: bytes)` | Constructs a Remote Mode command: 2-byte ASCII command + 2-byte little-endian length + arguments.                                |
    | `set_timer`             | Constructs the "TI" remote command to synchronize RTC by setting the current time.                                               |

- Context manager support: `with LprClient(...) as printer:`

#### Epson Remote Mode commands

`LprClient` can be used to send *Epson Remote Mode commands* to the printer. Notice that the LPR or RAW channels do not support receiving payload responses from the Epson printer.

Comprehensive, unified documentation for Epson’s *Remote Mode commands* does not exist: support varies by model, and command references are scattered across service manuals, programming guides and third-party sources (for example, the [Developer's Guide to Gutenprint](https://gimp-print.sourceforge.io/reference-html/x952.html) or [GIMP-Print - ESC/P2 Remote Mode Commands](http://osr507doc.xinuos.com/en/OSAdminG/OSAdminG_gimp/manual-html/gimpprint_37.html)).

General sequences:

| **Command**           | **Description**                                              | **String**   |
| --------------------- | ------------------------------------------------------------ |------------- |
| `LF`                  | Line Feed (new line).                                        |              |
| `SP`                  | Space.                                                       |              |
| `NUL`                 | b'\x00'.                                                     |              |
| `FF`                  | Form Feed (b'\x0c'); flushes the buffer / ejects the page.   |              |
| `INITIALIZE_PRINTER`  | Resets printer to default state (ESC @).                     |              |

Specific Epson sequences:

| **Command**           | **Description**                                              | **String**   |
| --------------------- | ------------------------------------------------------------ |------------- |
| `EXIT_PACKET_MODE`    | Exit IEEE 1284.4 (D4) packet mode. See [1], 5.1.1 Exit Packet Mode. Must be sent before any other command. |              |
| `REMOTE_MODE`         | Enter Epson Remote Command mode. See [1], 6.1.1 Enter Remote Mode  | `(R`         |
| `ENTER_REMOTE_MODE`   | Initialize printer and enter Epson Remote Command mode.      |              |
| `EXIT_REMOTE_MODE`    | Exits Remote Mode. See [1], 6.1.14 Terminate Remote Mode |              |
| `JOB_START`           | Begins a print job. "JS", b'\x00\x00\x00\x00'. See [1], Start job “JS” nn 00H 00H <job name> m1. It is necessary to send the TI command before the JS command. | `JS`         |
| `JOB_END`             | Ends a print job. "JE", b'\x00'. See [1], End job “JE” 01H 00H 00H | `JE`         |
| `PRINT_NOZZLE_CHECK`  | Triggers a nozzle check print pattern. "NC", b'\x00\x00'     | `NC`         |
| (nozzle check)        | "NC", b'\x00\x10'                                            | `NC`         |
| `VERSION_INFORMATION` | Requests firmware or printer version info. "VI", b'\x00\x00' | `VI`         |
| `LD`                  | Load NVR Settings. See [1], Load Power-On Default NVR into RAM (Remote Mode) "LD" 00H 00H | `LD`         |
| (Run print-head cleaning) | "CH" 02H 00H 00H                                         | `CH`         |
| (Return the printer ID) | ESC 01 @EJL [sp] ID\r\n | `ID` |

[1]: handbook named "EPSON Programming Guide For 4 Color EPSON Ink Jet Printer XP-410 (Level I)", also including the description of following remote mode commands:

Description    | Command | Syntax
-------------- | -- | ----------------------------------
Set printer timer, synchronize RTC | TI | "TI" 08H 00H 00H YYYY MM DD hh mm ss
Set horizontal print position | FP | “FP” 03H 00H 00H m1 m2
Turn printer state reply on/off | ST | “ST” 02H 00H 00H m1
Set Job Name | JH | Job name set “JH” nL nH 00H m1 m2 m3 m4 m5 <job name>
Paper Feed Setup | SN | Set mechanism sequence "SN" 01H 00H 00H
Set Media information | MI | Select paper media “MI” 04H 00H 00H m1 m2 m3
Set double paper print | DP | Select Duplex Printing “DP” 02H 00H 00H m1
Set user setting | US | User Setting “US” 03H 00H 00H m1 m2
Select paper path | PP | Select paper path “PP” 03H 00H 00H m1 m2
Save Setting | SV | “SV” 00H 00H

See https://gimp-print.sourceforge.io/reference-html/x952.html for other ones.

The following code prints the nozzle-check print pattern:

```python
from pyprintlpr import LprClient

with LprClient('192.168.1.100', port="LPR", queue='PASSTHRU') as lpr:
    data = (
        lpr.EXIT_PACKET_MODE +    # Exit packet mode
        lpr.ENTER_REMOTE_MODE +   # Engage remote mode commands
        lpr.PRINT_NOZZLE_CHECK +  # Issue nozzle-check print pattern
        lpr.EXIT_REMOTE_MODE +    # Disengage remote control
        lpr.JOB_END               # Mark maintenance job complete
    )
    lpr.send(data)
```

Some usage examples:

```cmd
python3 -m pylpr client -a 127.0.0.1 -f SECURITY.md -p lpr  # Send SECURITY.md to the LPR server using the default queue
python3 -m pylpr client -a 127.0.0.1 -p lpr -s user -q PASSTHRU  # List queue (compact form)
python3 -m pylpr client -a 127.0.0.1 -p lpr -S user -q PASSTHRU  # List queue (long form)
python3 -m pylpr client -a 127.0.0.1 -p lpr -P  # Traverse all jobs to be printed by the default queue
python3 -m pylpr client -a 127.0.0.1 -p lpr -R job  # Remove a job
```

### Server/Proxy API
Capture, forward, manage LPR requests. Also process RAW requests.

#### Basic Usage
```python
from pyprintlpr import LprServer

server = LprServer(save_files=True, save_path='lpr_jobs')
# The server can be integrated into an asyncio event loop for advanced use
# For simple usage, run via command line or module interface
```

#### LprServer Class Reference
- `LprServer(save_files=True, save_path=None)` – Initialize server

  - `save_files` (`bool`, default: `True`):
    If `True`, received print jobs (control and data files) are saved to disk. If `False`, jobs are discarded after dumping data.
  - `save_path` (`str`, default: `"lpr_jobs"`):
    if `save_files` is set to `True`, directory path where received print jobs are saved. If not specified, defaults to "lpr_jobs" in the current working directory.

- `get_next_job_id()` – Generate next job ID
- `parse_control_file(content: bytes)` – Parse LPR control file
- `format_queue_status(queue: str, long_format=False)` – Format queue status
- `save_job_files(job: LPRJob)` – Save job files to disk

## Examples

### Run the server:

```bash
python3 -m pyprintlpr server -a 192.168.1.100 -t -l 515
```

### Send print job:
```python
from pyprintlpr import LprClient

with LprClient('192.168.1.100', port=9100) as printer:
    printer.send(b'\x1B@Hello Printer!\x0C')
```

### Run as a module
```bash
python -m pyprintlpr client -a 192.168.1.100 -f file.txt
python -m pyprintlpr server -a 192.168.1.100 -t
```

## License

EUPL-1.2 License - See [LICENSE](LICENSE.txt) for details.
