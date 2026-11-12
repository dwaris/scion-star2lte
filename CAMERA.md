# SM-G965F camera validation

Tested on 12 November 2026 with iOS 16.5 (20F66), the `bringup-star2lte`
configuration, and the kernel commits below.

## Kernel prerequisites

Apply these `scion-kernel` commits in order to a tree containing the upstream
Star2 camera support:

```sh
git cherry-pick 8da59a8 a620a86 7e7dd4e 246263b
```

- `8da59a8 camera: fix switching delay`: close audio outputs left open by stop,
  allowing the capture graph to finish without its ten-second timeout.
- `a620a86 camera: fix star2lte telephoto`: select VC-only receiver mode for the
  S5K3M3 image and PDAF channels, and refresh telephoto availability in the bridge.
- `7e7dd4e camera: drop audio after output ends`: serialize sample delivery with
  output closure and discard samples arriving after the output ends.
- `246263b camera: ignore stale audio notifications`: discard format notifications
  from stopped audio sources and suppress duplicate end markers. Both could
  otherwise reach a consumer that the old capture graph had already released.

The tested bringup configuration enables all three cameras with these settings:

```text
CONFIG_CAM=5
CONFIG_CAM_S9=7
CONFIG_CAM_S9_REAR=y
CONFIG_CAM_S9_TELE=y
CONFIG_CAM_S9_TELE_PROBE=1
```

The boot-time telephoto probe lets the bridge select the real lens at 2x. These
settings require the kernel fixes above and the working Star2 bringup defaults;
enabling the cameras alone does not fix switching or gray telephoto frames.
This validation report does not change the device defaults.

## Results

| Check | Observation |
| --- | --- |
| Front/rear switching | Responsive; the ten-second graph shutdown timeout did not recur. |
| Rear 1x/2x | Both lenses produced real images; 2x no longer displayed gray buffer contents. |
| Continuous preview | One completed stream reported 5,901 frames, zero lost frames and zero FIFO errors. |
| Still photos | Two 4032x3024 images encoded successfully, in 7,728 ms and 10,445 ms. |
| Delayed audio work | Late samples, a late format notification and a duplicate end marker were caught. |
| Media-server stability | No media-server crash in the approximately five-minute capture window. |

Repeat front/rear and 1x/2x switching, take photos, then close Camera and leave
the phone running for at least two minutes. The earlier crashes occurred after
the switches; checking only the immediate preview missed them.

This validates the tested paths and interval, not long-term stability or every
camera mode. Video recording with microphone audio still needs an explicit
record/save/playback test. OIS is not driven, full ISP processing is absent, and
full-resolution photo encoding is slow.

## Other observations from the same run

- CloudKit repeatedly attempted setup without an iCloud account; missing services
  and local cloud-database errors were also logged. Early cloud-service disabling
  has not been implemented by these camera changes.
- ScreenTimeAgent aborted once after failing to load its local
  `RMAdminStore-Local.sqlite` store, then restarted. This was a separate service
  failure; the media server remained running.
- Repeated `AppleM2ScalerCSCDriver` lookup failures refer to absent Apple hardware.
  They produced substantial log noise, while the tested camera paths still worked.
- GPS/GNSS and cellular remain missing in this bringup. Bluetooth audio remains
  unverified; these are not failures established by the camera test itself.
