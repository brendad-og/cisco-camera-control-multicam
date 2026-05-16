# cisco_camera_control

A minimal toolset for controlling Cisco TTC8-07 / Precision HD 1080p cameras
without the original codec.

Reverse-engineered from MITM captures of a real codec ↔ camera session.
Works on cameras running firmware **HC9.15.0.11**.

## Contents

| File | Purpose |
|------|---------|
| `server.py` | HTTP API + web UI that drives a paired camera |
| `provision.py` | One-shot tool that "pairs" any reachable camera by replaying a captured codec session |
| `data/cppmf_session.bin` | 2.4 KB CPPMF (port 13491) management traffic — codec→camera |
| `data/doric_session.bin` | 42.6 MB DORIC (port 13496) firmware + identity upload — codec→camera |

## Quick start

### 1. Provision a camera (one-time, ~3 minutes)

If the camera was paired with a different codec, factory-reset, or you're not
sure — start here. After this it'll trust the codec identity baked into
`data/`.

```bash
./provision.py 10.0.0.92
# wait ~3 minutes; camera reboots and gets a new DHCP lease
```

The camera will likely come back on a **different IP** after rebooting
(its old DHCP lease lapses). Scan for it:

```bash
python3 -c "
import socket, concurrent.futures
def probe(i):
    s = socket.socket(); s.settimeout(0.5)
    try: s.connect(('10.0.0.'+str(i), 13496)); return '10.0.0.'+str(i)
    except: return None
with concurrent.futures.ThreadPoolExecutor(64) as ex:
    print([r for r in ex.map(probe, range(2,255)) if r])
"
```

### 2. Run the server

```bash
./server.py --camera 10.0.0.93
# open http://localhost:8080 in a browser
```

The web UI is a single page with pan/tilt/zoom number inputs. Hit "Go" to send
an absolute `PositionSet` command.

## HTTP API

| Endpoint | Method | Body | Purpose |
|----------|--------|------|---------|
| `/` | GET | — | Web UI |
| `/position` | POST | `{"pan": -10000..10000, "tilt": -2500..2500, "zoom": 1000..8000}` | Absolute position |
| `/status` | GET | — | `{"connected": bool, "last_pos": {...}}` |

`pan` and `tilt` units are XAPI (1/100 of a degree).
`zoom` is counterintuitive: lower = telephoto, higher = wide angle.

## How it works (brief)

The camera's `13496/tcp` is TLS with no certificate validation. After the TLS
handshake the codec sends a 143-byte DORIC banner containing its identity
("4 BLOBs"). The camera's response, the rest of the handshake, and the
command frames are all in this codebase.

`provision.py` replays a complete captured codec session — including ~42 MB
of firmware/peripheral package (`halley.pk`) — which:
1. Updates the camera's firmware to HC9.15.0.11 if needed
2. Installs the captured codec's identity as the camera's trust anchor

After provisioning, the camera will accept commands from `server.py` (which
sends the same identity blobs in its banner).

## Security note

The identity blobs in `server.py` and `data/*.bin` are **one specific codec's**
captured identity. Anyone with these files can control any camera ever
provisioned by this toolset. Treat the `data/` files as you would a private
key.
