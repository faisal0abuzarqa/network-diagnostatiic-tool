# Network Diagnostics Tool — Phase 1 Project Report

**Course:** 605346 — Information & Network Security Programming  
**University of Petra — Faculty of Information Technology — Department of Information Security**  
**Semester:** 1 (2026–2027)  

---

## 1. Introduction
This report presents the design, implementation, and testing of the **Network Diagnostics Tool**, developed as a four-member team project for Course 605346. The tool is a Python command-line client-server application capable of verifying host reachability (`PING`), measuring round-trip network latency (`ECHO`), and monitoring live server activity (`STATS`) over TCP sockets.

* **GitHub Repository Link:** `https://github.com/<your-username>/<your-repo-name>`
* **Team Members:**
  1. **Etaf Alomari** (`202511059`) — Server Implementation (`server`)
  2. **Muhannad Qawasmi** (`202420074`) — Reachability Client (`ping`)
  3. **Hala Jabr** (`202320571`) — Latency Client (`echo`)
  4. **Faisal Marwan** (`201910617`) — Server Activity Client (`stats`)

---

## 2. Background & Overview
Network diagnostics tools form the foundation of network administration and security monitoring. By actively probing services, engineers can distinguish between unreachable hosts, high-latency links, and overloaded servers. Implementing these diagnostics from scratch using low-level sockets demonstrates how application-layer protocols frame messages over transport-layer streams (TCP) and how concurrent servers safely manage shared state across multiple simultaneous clients without race conditions.

---

## 3. Steps to Run the Code
The application relies exclusively on Python's standard library (Python 3.9+) and uses `argparse` subcommands (`server`, `ping`, `echo`, `stats`):

1. **Launch the TCP Server:**
   ```bash
   python "TCP_Project_phase 1.py" server --host 127.0.0.1 --port 5000
   ```
2. **Run the Reachability Client (`PING`):**
   ```bash
   python "TCP_Project_phase 1.py" ping 127.0.0.1 5000
   ```
3. **Run the Latency Client (`ECHO`):**
   ```bash
   python "TCP_Project_phase 1.py" echo 127.0.0.1 5000 --count 4 --message "test"
   ```
4. **Run the Server Activity Client (`STATS`):**
   ```bash
   python "TCP_Project_phase 1.py" stats 127.0.0.1 5000 --repeat 4 --interval 1.5
   ```

---

## 4. Code Structure & Architecture Explanation
The project uses a unified newline-delimited JSON protocol over TCP (`SOCK_STREAM`):

* **Concurrent Server (`TCPServer` & `ClientHandler`):** Inherits from `socketserver.ThreadingTCPServer` with `daemon_threads = True` so each client connection runs in an isolated worker thread. `process_request()` validates JSON payloads and dispatches `PING`, `ECHO`, and `STATS` handlers. Shared state is protected using two `threading.Lock` instances: `counter_lock` guarantees atomic increments of `requests_count`, and `log_lock` ensures atomic, non-interleaved appends to a timestamped log file (`server_log_YYYYMMDD_HHMMSS.log`).
* **Reachability Client (`run_ping`):** Generates an 8-character UUID, sends `{"type": "PING", "id": req_id}`, verifies the `"PONG"` acknowledgment, and reports `Reachable` with RTT or `No response (timeout)` on socket timeout.
* **Latency Client (`run_echo`):** Sends `--count` sequential `ECHO` requests with unique identifiers (`attempt-uuid`), matches each reply to its request ID and payload, computes `Min`, `Avg`, and `Max` RTT using `time.perf_counter()`, and tracks timed-out attempts separately.
* **Server Activity Client (`run_stats`):** Queries `STATS` repeatedly (`--repeat` and `--interval`), validates `uptime_seconds` and `completed_requests`, formats uptime as `HH:MM:SS`, and displays live counter progression.
* **Validation & Exception Handling:** Custom `argparse` validators (`port_value`, `positive_int`, `finite_float`) reject invalid CLI inputs early, while network routines catch `socket.timeout`, `ConnectionRefusedError`, `socket.gaierror`, malformed JSON, and oversized payloads.

---

## 5. Testing & Findings
To verify concurrency and thread safety, the server was started on `127.0.0.1:5000` while all three clients ran simultaneously in separate terminals:
1. `stats` started polling every 1.5 seconds, initially reporting `completed_requests=1`.
2. `ping` verified reachability with an RTT of ~1.3 ms (`Reachable`), while `echo` sent 4 messages with an average RTT of ~1.25 ms and `0` timeouts.
3. Subsequent `stats` polls reflected the exact cumulative increase (`completed_requests=6`, then `7`), confirming zero race conditions in `counter_lock`.
4. Inspecting the generated `server_log_*.log` file confirmed that every request was recorded with its timestamp, worker thread name, client IP/port, request ID, and `OK` outcome without overlapping lines.

---

## 6. Team Member Contributions
* **Student 1 — Etaf Alomari (`202511059`):** Designed the JSON framing protocol, implemented the multithreaded TCP server (`TCPServer`, `ClientHandler`), and implemented thread-safe request counting and timestamped file logging.
* **Student 2 — Muhannad Qawasmi (`202420074`):** Implemented the `PING` reachability client, UUID acknowledgment matching, and timeout/connection error reporting.
* **Student 3 — Hala Jabr (`202320571`):** Implemented the `ECHO` latency client, per-request ID correlation, high-precision RTT statistics (`Min`/`Avg`/`Max`), and separate timeout tracking.
* **Student 4 — Faisal Marwan (`201910617`):** Implemented the `STATS` server activity client, uptime formatting, periodic polling (`--repeat`/`--interval`) for live demonstrations, and project documentation.