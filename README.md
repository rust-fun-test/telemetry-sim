# Telemetry Simulator

A pre-built binary that generates simulated bus telemetry and broadcasts it over **UDP multicast** every 10 ms. No configuration or arguments are required — run the binary and it starts sending.

## Transport

| Parameter | Value |
|-----------|-------|
| Protocol | UDP multicast |
| Multicast group | `224.0.0.1` |
| Port | `8904` |
| Interval | 10 ms |
| Format | `key=value, key=value` (one record per datagram, newline-terminated) |

## Binaries

| Platform | Path |
|----------|------|
| macOS (Apple Silicon) | `macos-arm64/telemetry-sim` |
| Linux (x86_64, musl) | `linux-musl/telemetry-sim` |
| Windows (x86_64) | `windows/telemetry-sim.exe` |

---

## Running the simulator

### macOS (Apple Silicon)

```bash
chmod +x macos-arm64/telemetry-sim
./macos-arm64/telemetry-sim
```

### Linux (x86_64, musl)

```bash
chmod +x linux-musl/telemetry-sim
./linux-musl/telemetry-sim
```

Works on any Linux distribution without glibc — the binary is statically linked via musl.

### Windows

```cmd
windows\telemetry-sim.exe
```

Or from PowerShell:

```powershell
.\windows\telemetry-sim.exe
```

On startup the simulator prints to stderr:

```
[telemetry-sim] Sending to multicast 224.0.0.1:8904 every 10ms
[telemetry-sim] Clients: join multicast group 224.0.0.1 on port 8904
```

---

## Receiving data

Clients must **join the multicast group** to receive datagrams. A plain `nc -u -l 8904` will not work.

### macOS / Linux (socat)

```bash
socat UDP4-RECVFROM:8904,ip-add-membership=224.0.0.1:127.0.0.1,reuseaddr,fork -
```

Run this in multiple terminals — each client receives every datagram independently.

### macOS / Linux / Windows (Python)

```bash
python3 -c "
import socket, struct
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('', 8904))
s.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP,
    struct.pack('4s4s', socket.inet_aton('224.0.0.1'), socket.inet_aton('127.0.0.1')))
while True:
    print(s.recv(512).decode(), end='', flush=True)
"
```

---

## Record types

One record type is chosen at random on each tick. Floating-point values are rounded to one decimal place.

### GPS Position

```
latitude=51.6, longitude=-0.3, speed=42.7, altitude=38.2
```

| Field | Range | Unit |
|-------|-------|------|
| `latitude` | -90.0 to 90.0 | decimal degrees |
| `longitude` | -180.0 to 180.0 | decimal degrees |
| `speed` | 0.0 to 120.0 | km/h |
| `altitude` | -100.0 to 5000.0 | metres |

### Door State

```
door=open
```

| Field | Values |
|-------|--------|
| `door` | `open`, `close` |

### Vehicle Telemetry

```
odometer=12483.5, fuel_consumption=8.3, turn_direction=left
```

| Field | Range / Values | Unit |
|-------|----------------|------|
| `odometer` | 0.0 to 999999.9 (monotonically increasing) | km |
| `fuel_consumption` | 3.0 to 20.0 | L/100km |
| `turn_direction` | `left` (20%), `right` (20%), `straight` (60%) | — |

### Passenger Counter

```
passengers_in=3, passengers_out=1, total_load=24
```

| Field | Range | Unit |
|-------|-------|------|
| `passengers_in` | 0 to 5 | count |
| `passengers_out` | 0 to min(5, total_load) | count |
| `total_load` | 0 to 100 (running total, clamped) | count |

---

## Developer task

Build a Rust application that connects to the telemetry feed, classifies each record, and writes parsed output as JSON — one text file per record type.

### Requirements

1. **Connect** to the UDP multicast group `224.0.0.1` on port `8904`.
2. **Read** datagrams continuously (each datagram is one record, newline-terminated).
3. **Parse** the `key=value, key=value` format into a structured representation.
4. **Classify** each record into one of four types based on its fields:
   - `gps` — contains `latitude`, `longitude`, `speed`, `altitude`
   - `door` — contains `door`
   - `vehicle` — contains `odometer`, `fuel_consumption`, `turn_direction`
   - `passenger` — contains `passengers_in`, `passengers_out`, `total_load`
5. **Serialize** each successfully parsed record to JSON and **append** it as a single line to the appropriate output file:
   - `gps.jsonl`
   - `door.jsonl`
   - `vehicle.jsonl`
   - `passenger.jsonl`

### Example JSON output

**gps.jsonl**
```json
{"latitude":51.6,"longitude":-0.3,"speed":42.7,"altitude":38.2}
```

**door.jsonl**
```json
{"door":"open"}
```

**vehicle.jsonl**
```json
{"odometer":12483.5,"fuel_consumption":8.3,"turn_direction":"left"}
```

**passenger.jsonl**
```json
{"passengers_in":3,"passengers_out":1,"total_load":24}
```

### Stretch goals

- Add a `--output-dir` flag to control where the `.jsonl` files are written.

### How to test

1. Start the simulator: `./macos-arm64/telemetry-sim` (or the binary for your OS).
2. In another terminal, start your Rust client.
3. Let both run for a few seconds, then inspect the four `.jsonl` files.
