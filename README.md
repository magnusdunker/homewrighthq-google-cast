# Google Cast for HomewrightHQ

An external integration for [HomewrightHQ](https://github.com/magnusdunker/homewrighthq) that controls Chromecasts, Google and Nest speakers and displays, Android TV and Cast groups as media players. It speaks the Cast protocol directly with the devices on the local network, bypassing Google's cloud.

## Installation

In HomewrightHQ: **Settings → Integrations → From GitHub**. Paste

```
https://github.com/magnusdunker/homewrighthq-google-cast
```

and press **Install**. Then add **Google Cast** under **Add integration**. The form shows the devices found on the network. If all are selected, devices that show up later are included too. Once the integration is installed, HomewrightHQ also suggests it on its own when it finds a Cast device.

There is only one setup, and it covers all devices. The choice of devices can be changed in the integration's settings.

### Devices that mDNS does not find

Devices are found with mDNS (`_googlecast._tcp`), which only works on the same network. If a device is on another VLAN, or HomewrightHQ runs in Docker without host networking, enter its address under **Ekstra adresser** (extra addresses), e.g. `192.168.20.15` or `tv.lan:8009`. Separate several addresses with commas. The address is tried when you save, and the integration tries again every 30 seconds if the device is gone at startup.

## What you get

Each device becomes a device in HomewrightHQ with one media player. The name is the one the device has in the Google Home app.

The state is:

- **off**: no app is running, or the device shows its screensaver
- **idle**: an app is running but not playing anything
- **playing**, **paused** and **buffering** according to the media's state

The attributes follow Home Assistant: `volume_level`, `is_volume_muted`, `app_id`, `app_name`, `media_title`, `media_artist`, `media_album_name`, `media_series_title`, `media_content_id`, `media_content_type`, `media_duration`, `media_position`, `media_position_updated_at`, `entity_picture` and `supported_features`.

The services are:

- `turn_on` starts the Default Media Receiver, and `turn_off` stops the running app
- `volume_set`, `volume_mute`, `volume_up` and `volume_down`
- `media_play`, `media_pause`, `media_play_pause`, `media_stop` and `media_seek`
- `media_next_track` and `media_previous_track`, when the app has a queue
- `play_media` plays a URL in the Default Media Receiver:

```yaml
action: media_player.play_media
target:
  entity_id: media_player.stue
data:
  media_content_id: http://nas.lan/musik/sang.mp3
  media_content_type: music        # or a MIME type such as audio/mpeg
  extra:
    title: Sang
    thumb: http://nas.lan/musik/cover.jpg
    stream_type: LIVE              # for radio and other live streams
```

If `media_content_type` is not a MIME type, it is guessed from the URL's file extension. The service only responds once the device has loaded the media, and fails if the device cannot play it.

The integration keeps a connection open to each device, and the devices send their state themselves when it changes. If the connection is lost, the player becomes unavailable, and the integration reconnects after 1 second and then with longer pauses up to 30 seconds. If the device changes IP address, mDNS detects it.

The integration remembers the address mDNS last gave each device, and connects to it at startup. That way a device is included even if it does not answer mDNS when HomewrightHQ starts. It also asks on mDNS again every 5 minutes, so a device that only answers later is found.

## Development

Built in Rust on the `thehome-plugin` SDK from [thehome-example-integration-rust](https://github.com/magnusdunker/homewrighthq-example-integration-rust).

- `src/proto.rs`: Cast messages (protobuf) and the framing around them
- `src/conn.rs`: TLS to the devices, which have self-signed certificates
- `src/device.rs`: the connection to one device, the services and reconnecting
- `src/status.rs`: from the device's status to state and attributes
- `src/discovery.rs`: mDNS and the device's `eureka_info` page
- `src/lib.rs`: the setup and the connection to HomewrightHQ
- `tests/`: tests against a fake Cast device in `tests/fake` and a fake HomewrightHQ in `tests/host`

```
cargo test
cargo build --release
```

A new version is released with a tag, e.g. `git tag v0.2.0 && git push origin v0.2.0`. The workflow builds for Windows, Linux (x86_64 and arm64, static musl) and macOS and puts the binaries on the release.

## License

MIT
