# f300 hardware validation

Date: 2026-10-30. Camera: f300; firmware: `v5.1.10.1811061306`. The camera was identified in the MIPC application and tested directly over its LAN HTTP interface. Local login used the device serial and its separate Device/Admin PW.

**30 of 31 checks passed; one endpoint is unsupported on this firmware.** No persistent settings, passwords, network configuration, storage contents, or firmware were changed by these tests. Event queues and media sessions were created explicitly for testing and closed afterwards.

## Read-only checks

| Check | Result |
| --- | --- |
| `login` | Passed |
| `identity` | Passed |
| `device_info` | Passed |
| `microphone_inputs` | Passed |
| `microphones` | Passed |
| `speaker_outputs` | MipcUnsupportedError (http_404) |
| `speaker` | Passed |
| `imaging` | Passed |
| `misc` | Passed |
| `osd` | Passed |
| `alert_config` | Passed |
| `alert_actions` | Passed |
| `motion_mask` | Passed |
| `recording` | Passed |
| `disks` | Passed |
| `video_sources` | Passed |
| `network_without_scan` | Passed |
| `ntp` | Passed |
| `date` | Passed |
| `cruise` | Passed |
| `scenes` | Passed |
| `schedule` | Passed |
| `message_filters` | Passed |
| `messages` | Passed |
| `snapshot` | Passed |
| `live_stream` | Passed |
| `session_check` | Passed |
| `event_queue_create` | Passed |
| `event_poll` | Passed |
| `event_queue_destroy` | Passed |
| `logout` | Passed |

## Media verification

* Snapshot: a valid JPEG was decoded/verified in memory at 1920×1080. No image was saved or published.
* Streaming: the library negotiated an RTMP URL naming the configured camera. An external PyAV decoder opened that URL and decoded three H.264 video frames at 1920×1080. PyAV is a test tool, not a library dependency.
* No stream URLs, credentials, session tokens, device serials, MAC addresses, network configuration values, video frames or audio recordings are included in this report.

## Device-specific findings

* `ccm_speakers_get` returns HTTP 404. `ccm_speaker_get` succeeds and returns speaker configuration. The library now raises `MipcUnsupportedError` for HTTP 404/405/501 instead of reporting a network failure. Discovery failure must not be treated as proof that the camera has no speaker.
* The microphone `entity` field is a nested object on this firmware. Its type annotation now accommodates that shape, and a synthetic regression fixture verifies preserving the nested object when encoding a settings update.
* The local device username is its serial. MIPC cloud account credentials are separate from the device credentials and do not authenticate this local interface.
* Queue creation, subscription, a two-second empty poll and queue destruction succeeded. No events were received during that poll; actual motion-event delivery has not been demonstrated.

## Still unverified

Settings writes and read/modify/write persistence; PTZ/physical movement; storage maintenance; firmware updates; reboot/restore; password changes; pairing/external accessories; gateway/box operations; recorded-video playback; talk audio upload; RTSP; and compatibility with other models or firmware. Readable cruise/schedule/scene documents do not establish support for their associated actions.

The test checked direct local operation. It did not establish that the camera firmware makes no independent external connections. The Windows application was used for identifying the device and reading connection details; library RPCs and media decoding targeted the local camera.
