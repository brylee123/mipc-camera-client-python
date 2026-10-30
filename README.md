# MIPC camera client — local control

A Python client that talks directly to a MIPC-compatible camera or gateway at an address you provide. No Windows application, vendor JavaScript, native plugin, cloud account discovery, telemetry, or background service is included.

This fork expands the original `pan-maruda/mipc-camera-client-python` library. The development branch is **0.2.0.dev0**, not a published PyPI release. The Windows client supplies protocol evidence; camera compatibility still needs hardware testing.

## Install this review branch

Python 3.10 or newer:

```shell
python -m pip install "mipc-camera-client @ git+https://github.com/brylee123/mipc-camera-client-python.git@feature/typed-device-settings"
```

## Basic use

```python
import os
from pathlib import Path
from mipc_camera_client import MipcCameraClient

with MipcCameraClient("192.168.1.50:80", timeout=10) as camera:
    camera.login(os.environ["CAMERA_USER"], os.environ["CAMERA_PASSWORD"])
    print(camera.get_info())
    Path("snapshot.jpg").write_bytes(camera.get_image())
    print(camera.get_stream_url())  # URL negotiation only; caller owns playback
```

Construction does not contact anything. Login uses the existing MIPC DH/DES protocol at the supplied host. Each operation stays at that host, ignores environment proxies and .netrc, rejects HTTP redirects, and has a bounded timeout. There is no automatic login retry or cloud fallback. The legacy protocol uses HTTP and legacy cryptography; it does not provide modern transport security.

The hardware-tested f300 uses its device serial as the local login name and its separate Device/Admin PW. For that model, use `camera.login(camera.get_device_sn(), device_password)`. MIPC cloud account credentials do not authenticate the local camera. Other firmware may use a different local username.

## Settings and controls

```python
# After login; call only the settings your hardware supports.
print(camera.get_microphone_inputs())
print(camera.get_speaker_outputs())
print(camera.get_imaging())
camera.update_imaging(brightness=55, mode="auto")
camera.set_microphone_level(60)
camera.update_microphone("mic0", silent=True)
camera.set_speaker_level(40)
camera.set_flip(True)
camera.update_osd(text="Front door", date={"time_enable": 1})
print(camera.get_alert_config())
print(camera.get_recording())
print(camera.get_disks())
camera.control_ptz(tilt_x=5, tilt_y=0)
```

Document update methods fetch the current document and preserve unrelated and unknown fields. They reject fields not returned by the camera. Simple setters validate their input ranges; firmware-specific values in advanced document updates are passed without guessed enum conversions. Array replacement is explicit, not an automatic merge of schedules or actions. Use the input/output tokens returned by your device; `mic0`, `ao0`, `vs0`, `p0`, and `ptz0` are examples/defaults, not universal capabilities.

The API covers audio inputs/outputs, image modes and levels, flip/power-frequency configuration, OSD, motion and alert configuration/actions, recording schedules, storage, date/NTP, network/Wi-Fi, PTZ/cruise, scenes, schedules, external accessories, gateway/box operations, device passwords, snapshots, media negotiation, events, and maintenance. See [the method reference](docs/API.md).

## Capability inspection

```python
report = camera.probe_capabilities(["imaging", "microphones", "osd", "disks"])
print(report.readable)
print(report.rejected)
```

Probing is opt-in and issues only the selected reads. A rejected read can mean missing support, insufficient permissions, or a different response schema. A successful read does not prove that writes are supported. Transport and session failures propagate immediately.

## Local event queues

```python
with camera.event_queue() as events:
    for item in events.poll(timeout=5):
        print(item)
```

Queues use the observed `mmq_create`, `ccm_subscribe`, `mmq_pick`, and `mmq_destroy` flow. Wire timeouts are milliseconds; Python timeouts are seconds. Polling is synchronous and blocks other operations on the same client. Use a separately logged-in client when you need simultaneous controls. The library starts no threads and does not reconnect silently. Event delivery and queue lifetime behavior still require camera testing.

## Advanced endpoints

The data-only [protocol inventory](docs/PROTOCOL.md) records **81 observed RPCs: 72 device/queue operations, 5 authentication operations, and 4 account operations**. It includes bundle hashes, call-site offsets, observed field names, and request shapes in `mipc_camera_client/endpoints.json`. No vendor bundle or cached camera/account payload is shipped.

```python
# Structured Python input; session IDs and MIPC's HTTP encoding are added.
print(camera.call("ccm_msg_get", flag=0, start=0, counts=20, filter="record"))
```

`call()` accepts catalogued device/queue methods. Account registration, recovery and binding are excluded. `run_rpc()` remains the original low-level escape hatch for flattened parameters at the configured host. Library exceptions identify the method without including credential-bearing URLs or response dumps.

Maintenance operations execute when explicitly called. `reboot()`, `restore()`, `control_storage("format")`, `set_network()`, and password changes can disrupt the device or erase data. Nothing calls them automatically. Firmware checks/downloads, NTP, alert actions and network configuration may cause the **camera firmware** to contact other hosts; the library itself does not follow those hosts. Complete offline isolation requires camera network restrictions.

Stream, playback and talk methods negotiate URLs; they do not implement a player or the native plugin's audio encoding/upload pipeline. Returned media URLs must match the configured camera hostname unless `allow_remote=True` is explicitly requested. RTMP/RTSP support depends on firmware. Native plugin clip export and full Windows UI parity are not claimed.

## Command line

Set `CAMERA_HOST`, `CAMERA_USER` and `CAMERA_PASSWORD` in your environment, then:

```shell
mipc_camera_client info
mipc_camera_client read imaging
mipc_camera_client probe
mipc_camera_client snapshot image.jpg
mipc_camera_client stream
mipc_camera_client ptz --x 5 --y 0
mipc_camera_client endpoints ccm_osd_set
mipc_camera_client rpc ccm_msg_get --data-file request.json
```

`endpoints` works offline without credentials. RPC input is a JSON file or standard input (`--data-file -`). It uses structured fields, not flattened `d...` names. RPC commands can write settings and execute immediately. Output documents can contain device identifiers or tokens; treat saved output accordingly.

## Development and verification

```shell
python -m pip install poetry
poetry install --with dev
poetry run pytest
poetry run ruff check mipc_camera_client tests tools
poetry run ruff format --check mipc_camera_client tests tools
poetry build
```

Tests cover upstream crypto vectors, request encoding, all catalog device dispatch entries, conservative settings updates, auth/error handling, media host restrictions, queue lifecycle, and a real loopback HTTP camera simulator. A read-only f300 hardware run also passed 30 checks, produced a valid 1920×1080 JPEG and decoded three local H.264/RTMP video frames. Speaker-list discovery returned HTTP 404 while direct speaker configuration remained readable; this is now reported as `MipcUnsupportedError`. See [hardware validation](docs/HARDWARE.md) for scope and limits. Settings writes and maintenance operations have not been hardware-tested.

BSD-3-Clause. Original protocol implementation and attribution retained in [LICENSE](LICENSE).
