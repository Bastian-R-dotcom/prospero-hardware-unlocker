# Prospero Hardware Unlock

Prospero Hardware Unlock is an ELF payload for the standard PS5. It lets you request the highest CPU performance state (**P0**) and set a GPU clock target in MHz while a native PS5 game is running. The configuration and status readings are available through the console's FTP server.

## Requirements

- A standard PS5 with a working ELF loader.
- A running FTP server on the console with access to `/data`.
- A native PS5 game with a title ID starting with `PPSA`, such as GTA 6.

## Warning

- Use Prospero Hardware Unlock at your own risk. Its developers are not responsible for any damage resulting from the use of the payload.

- We recommend using a custom fan curve while the payload is active during gameplay.

## Quick start

1. Import `prospero-hardware-unlock_v1.1.elf` into Payload Manager or launch it with your ELF loader. Keep the `_v1.1` suffix so Payload Manager can display the version. If an older version is running, stop it before starting the new one.
2. Open `/data/hardware-unlock/settings.txt` over FTP. On first launch, the ELF creates the file with all settings set to `0`. Later launches use the saved settings.
3. To enable automatic CPU retries and request **2,230 MHz** for the GPU, save the settings below and upload the file back to the console. In FileZilla, select **Transfer → Transfer type → Binary**.
4. Start the game or return to it if it is already running. Open `/data/hardware-unlock/status.txt` over FTP to check the payload execution result.

```ini
cpu_boost=1
cpu_auto=1
gpu_mhz=2230
```

The running process is named `pldhrdunlock.elf`.

## Configuration

| Setting | Values | Effect |
| --- | --- | --- |
| `cpu_boost` | `0` or `1` | `1` requests P0 for cores in another P-state. `0` disables CPU requests. |
| `cpu_auto` | `0` or `1` | When `cpu_boost=1`, setting this to `1` enables automatic retries. This setting is optional and defaults to `0`. |
| `gpu_mhz` | `0`, `1800` to `2200` in 50 MHz steps, or `2230` | Sends the target frequency to the GPU driver. `0` disables GPU requests. |

`cpu_boost` and `gpu_mhz` are required. The file supports comments starting with `#` and Windows line endings.

The ELF applies the settings to a newly detected PS5 game as soon as valid settings are available. It also applies them after accepting an edit or re-upload during gameplay.

P0 selects a CPU performance state. `gpu_mhz` sets a GPU clock target. The firmware continues to manage the actual clock speeds based on load and the available power budget.

## Disabling requests

Set `cpu_boost`, `cpu_auto`, and `gpu_mhz` to `0`. To restore the original clock profile, save these settings and restart the console.

## Notifications

| Event | PS5 notification |
| --- | --- |
| Payload starts | `Prospero Hardware Unlock v1.1 is starting.` followed by `By @Bastian-R-dotcom` on the next line. |
| Native PS5 game detected | `Prospero Hardware Unlock detected PS5 game PPSA38500.` The message includes the detected title ID. |
| Settings update accepted | `Prospero Hardware Unlock settings updated.` |
| Settings file fails validation | `Prospero Hardware Unlock rejected settings.txt.` followed by the validation error on the next line. |

The payload sends the game notification once per newly detected game session, including when a game is already running at startup.

## Polling

Settings, game detection, and hardware readings share one polling loop. The first pass runs at startup. The ELF waits **3 seconds** after each pass, so the time between passes also includes file and driver operations.

| Operation | Timing |
| --- | --- |
| `settings.txt` | Read at the start of each pass. Validation requires two consecutive matching reads. A stable edit is usually validated **3–6 seconds** after the FTP transfer finishes. |
| Native PS5 game detection | Checked each pass after reading `settings.txt`. |
| GPU clock request (`gpu_mhz`) | With a nonzero target, the ELF attempts one request when it first applies valid settings to a detected game. Each accepted edit or re-upload of `settings.txt` during gameplay triggers another request in the same pass. |
| GPU clock reading (`gpu_reported_mhz`) | Sampled on every pass while a native PS5 game is detected, about **3 seconds** plus processing time between readings. Sampling continues during the CPU retry cooldown. |
| CPU clock and P-state readings | All eight cores are sampled on the same pass as the GPU clock. |
| `status.txt` | Checked at the end of each pass and rewritten when its contents change. |

### Automatic CPU retries

With `cpu_boost=1` and `cpu_auto=1`, retries use the CPU readings from the polling loop. The ELF needs **four consecutive samples** with at least one core outside P0 before it can retry. Each retry starts a new count of four samples and the timer enforces a **5-second minimum** between attempts. With the usual polling interval, persistent non-P0 readings lead to retries about **12 seconds** apart.

A retry can request P0 for multiple cores and counts as one attempt even if the driver returns an error. After **10 automatic retry attempts**, the ELF waits **5 minutes** before it can start another batch. It retries on the first eligible pass after the cooldown.

Settings checks, CPU readings, and `status.txt` updates also continue during the cooldown. A valid settings reload can still apply its CPU and GPU clock requests. The retry schedule carries over when you reload the settings or switch games during the same payload session.

## Reading `status.txt`

Download a fresh copy of `status.txt` over FTP to see the latest values.

| Field | Meaning |
| --- | --- |
| `hardware_unlock` | Payload version, shown as `1.1` for this release. |
| `game` | Detected game title ID. `waiting_for_ps5_game` means the ELF is waiting for a native PS5 game. |
| `settings_status` | `ready` means the ELF has validated the settings. `waiting_for_stable_file` means it is waiting for a matching read. Check `settings_error` if the state is `invalid` or `unavailable`. |
| `cpu_result` | `already_p0` means all eight cores were already in P0 during the initial check. `request_accepted` means the driver accepted the requests it received. |
| `cpu_p0_requests` | Number of accepted per-core P0 requests when the settings were applied. The ELF skips cores already in P0. |
| `cpu_auto_state` | `all_p0` means all eight cores were in P0 in the latest reading. `observing` means the ELF is collecting readings or waiting to retry. `cooldown` means the five-minute pause is active. |
| `cpu_auto_result` and `cpu_auto_p0_requests` | Result of the latest automatic retry and the number of per-core P0 requests accepted in that retry. |
| `cpu_core_N_pstate` and `cpu_core_N_reported_mhz` | P-state and clock speed reported by the driver for CPU core `N` (0 to 7). A P-state of `0` means P0. |
| `gpu_requested_mhz` and `gpu_reported_mhz` | Configured GPU clock target and reported GPU clock speed. The two values can differ. `gpu_result=request_accepted` confirms that the driver accepted the target. |

If a driver request fails, check the available `*_request_errno` fields. `partial_driver_error` means the driver accepted requests for some cores and rejected at least one. For a core-specific error, `cpu_error_core` or `cpu_auto_error_core` identifies the first affected core.

## Compatibility and testing

- **v1.0 testing** — With v1.0, the driver accepted both CPU and GPU requests while GTA 6 was running on a standard PS5. The firmware version tested was 13.60.
- **v1.1 testing** — Tests with a simulated PS5 covered the 2,230 MHz target, process naming, notification triggers, polling, P0 retries, automatic game detection.

## License and credits

This project is licensed under **GPL-3.0-only**. [License text](LICENSE).

Thanks to Bastian-R-dotcom for adding CPU and GPU monitoring, notification triggers, polling, P0 retries, support for native PS5 games and automatic game detection. [GitHub profile](https://github.com/Bastian-R-dotcom).

Thanks to gasb07 and the NeoEnabler contributors for the original clock-control commands. [NeoEnabler](https://github.com/gasb07/NeoEnabler).

Thanks to PSDevWiki for its PS5 documentation. [PSDevWiki](https://www.psdevwiki.com/).
