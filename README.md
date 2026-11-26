# Prospero Hardware Unlock

Prospero Hardware Unlock PS5 is an ELF payload for the standard PS5. It lets you request the highest CPU performance state (**P0**) and set a GPU clock target in MHz while a native PS5 game is running. The configuration and status readings are available through the console's FTP server.

## Requirements

- A standard PS5 with a working ELF loader.
- A running FTP server on the console with access to `/data`.
- A native PS5 game with a title ID starting with `PPSA`, such as GTA 6.

## Warning

- Use Prospero Hardware Unlock at your own risk. Its developers are not responsible for any damage resulting from the use of the payload or its source code.

- We recommend using a custom fan curve while the payload is active during gameplay.

## Quick start

1. Launch `prospero-hardware-unlock.elf` with your ELF loader. If an older version is running, stop it before starting the new one.
2. Open `/data/prospero-hardware-unlock/settings.txt` over FTP. On first launch, the ELF creates the file with all settings set to `0`. Later launches use the saved settings.
3. To enable automatic CPU retries and clock request for the GPU, save the settings below and upload the file back to the console. In FileZilla, select **Transfer → Transfer type → Binary**.
4. Start the game or return to it if it is already running. Open `/data/prospero-hardware-unlock/status.txt` over FTP to check the result.

```ini
cpu_boost=1
cpu_auto=1
gpu_mhz=2200
```

## Configuration

| Setting | Values | Effect |
| --- | --- | --- |
| `cpu_boost` | `0` or `1` | `1` requests P0 for cores in another P-state. `0` disables CPU requests. |
| `cpu_auto` | `0` or `1` | When `cpu_boost=1`, setting this to `1` enables automatic retries. This setting is optional and defaults to `0`. |
| `gpu_mhz` | `0`, or `1800` to `2200` in 50 MHz steps | Sends the target frequency to the GPU driver. `0` disables GPU requests. |

`cpu_boost` and `gpu_mhz` are required. The file supports comments starting with `#` and Windows line endings.

The ELF checks the file about once per second and waits for two identical reads before applying a new version. It sends the initial requests when it detects a PS5 game or when a valid version of `settings.txt` is uploaded again. It sends the GPU target once each time it applies the settings.

With `cpu_auto=1`, the ELF monitors all eight cores. If four consecutive readings show at least one core outside P0, it requests P0 again for the affected cores. After three retries, it waits for ten consecutive readings with all eight cores in P0, a new valid settings upload, or a different game. If the driver rejects a retry, automatic retries pause until the settings are updated or a different game starts.

P0 selects a CPU performance state. `gpu_mhz` sets a GPU clock target. The firmware continues to manage the actual clock speeds based on load and the available power budget.

## Reading `status.txt`

While a detected game is running, the ELF samples clock speeds and P-states about once per second. CPU monitoring uses the same readings. The ELF rewrites the file when its contents change. Download a fresh copy over FTP to see the latest values.

| Field | Meaning |
| --- | --- |
| `game` | Detected game title ID. `waiting_for_ps5_game` means the ELF is waiting for a native PS5 game. |
| `settings_status` | `ready` means the ELF has read and validated the settings. Check `settings_error` if validation fails. |
| `cpu_result` | `already_p0` means all eight cores were already in P0 during the initial check. `request_accepted` means the driver accepted the requests it received. |
| `cpu_p0_requests` | Number of cores for which the driver accepted a P0 request when the settings were first applied. |
| `cpu_auto_state` | `all_p0` means all eight cores were in P0 in the latest reading. `retry_limit` means the three-retry limit was reached. |
| `cpu_auto_retries` and `cpu_auto_result` | Number of retries since the last sustained period in P0 and result of the most recent retry. |
| `cpu_core_N_pstate` and `cpu_core_N_reported_mhz` | P-state and clock speed reported by the driver for CPU core `N`. |
| `gpu_requested_mhz` and `gpu_reported_mhz` | Configured GPU clock target and reported GPU clock speed. `gpu_result=request_accepted` confirms that the driver accepted the target. |

If the driver rejects a request, check the available `*_request_errno` fields. `partial_driver_error` means the driver accepted some CPU requests before rejecting a subsequent one.

## Disabling requests

Set `cpu_boost`, `cpu_auto`, and `gpu_mhz` to `0`. To restore the original clock profile, save these settings and restart the console.

## Compatibility and testing

- **Console test** — With v1.0, the driver accepted both CPU and GPU requests while GTA 6 was running on a standard PS5. The firmware version tested was 13.60.

## License and credits

This project is licensed under **GPL-3.0-only**. [License text](LICENSE).


Thanks to Bastian-r-dotcom [Profile](https://github.com/Bastian-R-dotcom) for adding cpu/gpu monitoring and automatic retries and adapting the payload to work with native PS5 games.
Thanks to gasb07 and the [NeoEnabler](https://github.com/gasb07/NeoEnabler) contributors, and to [PSDevWiki](https://www.psdevwiki.com/) for its PS5 documentation.



