# MIPC Camera for Home Assistant — alpha

## Setup
1. Download \`custom_components/mipc_camera/\` to your HA configuration's
   \`custom_components/mipc_camera/\` directory.
2. Restart Home Assistant.
3. Go to **Settings > Devices & services > Add integration > MIPC Camera**.
4. Enter the camera's LAN host, username, and password.
5. Configure PTZ/stream availability under the integration's options.

## Entities

- Camera: JPEG snapshot; optional RTMP stream via HA's FFmpeg/stream pipeline.
- Buttons: pan left/right and tilt up/down (optional); movement directions and
  step values must be verified on each model.
- Diagnostics: connection status, serial number, last seen.
- Action: \`mipc_camera.move\` takes \`config_entry_id\`, relative \`x\` and
  \`y\` values, optionally \`pan_speed\` and \`tilt_speed\`.

## Privacy and protocol notes

RTMP playback can be refused when the URL points to a nonlocal address.
\`allow_remote_stream\` is off by default. Enabling it may send camera media
through an untrusted service. The HTTP RPC itself uses the upstream
unencrypted \`http://HOST/ccm/\` interface. Use a trusted VLAN/LAN only.

The library currently implements no runtime camera capability query.
PTZ controls can be disabled for fixed cameras. The four MIPC stream tokens
(\`p0\` through \`p3\`) are passed to the camera; availability and quality
are hardware-dependent.

## Known limitations

- No supported ONVIF discovery, RTSP transcoding, recording toggles, SD card
  controls, motion detection, night mode, privacy mode, or audio settings.
- The client does not currently support auto-relogin after session expiry.
- Native RTMP requires Home Assistant's stream stack/FFmpeg to accept the URL.
- Snapshot, stream, and PTZ have not been exercised against real hardware.
- Client 0.1.3 lacks timeouts; the adapter wraps the private requests session
  with a per-instance 10-second timeout. Remove when upstream adds timeouts.
- A camera may interpret relative PTZ movement differently.

## Dependency strategy

Prefer keeping the Python client library in
\`brylee123/mipc-camera-client-python\` and moving this integration into a
**separate** HACS repository before publishing releases. This repo's existing
GitHub release workflow publishes to PyPI on releases and MUST NOT be used
to release the HA integration in-place.
