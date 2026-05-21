# Vehicle_Paralyed

# AutoNexus-X

AutoNexus-X is a Flutter app with two independent modules:

- `Drive Control` for BLE-based vehicle commands
- `Driver Monitor` for camera-based drowsiness detection

The BLE logic stays isolated from camera and ML processing. The drowsiness
monitor uses Google ML Kit face detection with eye-open probabilities from the
same ESP32-S3 camera board, and it forwards alert commands back to that board.

## BLE and ESP32

The phone app does not connect to `COM6` or `COM7` directly. The USB serial port
is only your laptop-side programming/debug link. The app connects to the ESP32
over BLE.

This project now scans for ESP32 boards that either:

- advertise one of these names: `ESP32_DRIVE`, `ESP32`, `AutoNexus-X`, `AutoNexus_X`
- expose the Nordic UART service UUID `6e400001-b5a3-f393-e0a9-e50e24dcca9e`

BLE command surface:

| Function | Command |
| --- | --- |
| Door lock | `LOCK` |
| Door unlock | `UNLOCK` |
| Ramp in start | `RAMP_IN_START` |
| Ramp out start | `RAMP_OUT_START` |
| Ramp stop | `RAMP_STOP` |
| Ramp latch lock | `RAMP_LOCK` |
| Ramp latch unlock | `RAMP_UNLOCK` |
| Driver drowsiness buzzer start | `ALERT_SLEEP_START` |
| Driver drowsiness relay on | `ALERT_RELAY_ON` |
| Driver drowsiness clear | `ALERT_CLEAR` |

## Driver Monitor rules

The Driver Monitor module:

- uses the same ESP32-S3 camera board at `http://192.168.4.1/capture`
- uses Google ML Kit face detection
- reads `leftEyeOpenProbability` and `rightEyeOpenProbability`
- treats both eyes as closed when both values are below `0.3`
- ignores short blinks with a small delay filter
- starts a timer when both eyes stay closed
- resets the timer when eyes reopen or no face is detected
- triggers audio, vibration, and on-screen warning when the threshold is exceeded
- rings the ESP32 buzzer for the first `5` seconds after the threshold is crossed
- turns on the ESP32 relay if the eyes stay closed after those `5` seconds

Default threshold: `5` seconds

Configurable threshold range: `2` to `10` seconds

## ESP32 wiring profile

Your single ESP32-S3 camera board should map commands to this hardware profile:

| Component | Signal | ESP32 pin |
| --- | --- | --- |
| Camera module | Data and control | `GPIO 4-18` |
| Buzzer | Alert output | `GPIO 2` |
| Relay | Alert escalation output | `GPIO 45` |
| Ramp motor | RPWM | `GPIO 39` |
| Ramp motor | LPWM | `GPIO 40` |
| Shared motor enable | ENABLE | `GPIO 47` |
| Wheelchair lock | RPWM | `GPIO 48` |
| Wheelchair lock | LPWM | `GPIO 1` |

Suggested mapping:

- Door lock stays visible in the app but does not drive hardware on this single-board profile
- Ramp motor uses `GPIO 39 / GPIO 40`
- Wheelchair lock uses `GPIO 48 / GPIO 1`
- Buzzer uses `GPIO 2`
- Relay uses `GPIO 45`

## Dependencies

Core packages in use:

- `flutter_blue_plus`
- `provider`
- `permission_handler`
- `google_mlkit_face_detection`
- `http`
- `path_provider`
- `audioplayers`
- `vibration`

## File structure

```text
lib/
  main.dart
  providers/
    control_provider.dart
  screens/
    drive_control_screen.dart
    driver_monitor_screen.dart
    home_screen.dart
  services/
    ai_monitor_service.dart
    bluetooth_service.dart
    permission_helper.dart
  widgets/
    connection_status_bar.dart
    hardware_profile_section.dart
    lock_control_section.dart
    ramp_control_section.dart
assets/
  audio/
    alert.wav
```

## Setup

1. Install Flutter dependencies:

```bash
flutter pub get
```

2. Make sure your ESP32 firmware advertises BLE with the NUS UUID above.

3. Run on a real Android phone:

```bash
flutter run
```

4. Grant Bluetooth permissions when prompted.

5. Connect the phone to the ESP32-S3 camera board Wi-Fi AP:
   `AutoNexus-X-CAM`

6. Open `Driver Monitor` and use the default source:
   `http://192.168.4.1`

## Build

Release APK output:

`build/app/outputs/flutter-apk/app-release.apk`

Build command:

```bash
flutter build apk --release
```
