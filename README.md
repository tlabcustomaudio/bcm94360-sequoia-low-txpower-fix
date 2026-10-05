# BCM94360 on macOS Sonoma/Sequoia (OCLP): Wi‑Fi sees no networks — the card transmits at ~3 dBm

A fix for a native Apple Broadcom card that "works" after the usual OCLP Modern Wireless setup (driver loads, Wi‑Fi turns on) but **finds no networks**, or finds some and **never stays connected**, while the same card works fine in Windows.

The common advice is *"BCM94360 cards don't need any Fixup kext"*. In this case one was needed: the driver came up with country code `X2` and a TX power table of about **3 dBm**. Access points couldn't hear the card. **AirportBrcmFixup with `brcmfx-country=#a`** brought power back and the card joined at once.

## Tested setup

| | |
|---|---|
| Card | Apple **BCM94360CS2** (`14e4:43a0`, subsystem `106b:0117`, 2 chains), on a passive PCIe x1 adapter, 2 antennas |
| macOS | 15.7.7 Sequoia (24G720) |
| OpenCore | 1.0.7 (also fine after updating to 1.0.8) |
| OCLP | 2.4.1, Modern Wireless root patch applied |
| SMBIOS | iMac19,1 (desktop Coffee Lake) |
| Already in place | `IOSkywalkFamily.kext` + Block `com.apple.iokit.IOSkywalkFamily`, `IO80211FamilyLegacy.kext` + `AirPortBrcmNIC.kext` plugin (MinKernel 23.0.0), `AMFIPass.kext`, `csr-active-config` `03080000`, `SecureBootModel` Disabled |

Under Ventura (native driver, no OCLP) the card had worked, weakly. It stopped after upgrading to Sonoma/Sequoia.

## Symptoms

- Wi‑Fi is on, but the menu bar lists **no networks**. `system_profiler SPAirPortDataType` lists none and a CoreWLAN scan returns 0.
- `networksetup -setairportnetwork` → `Could not find network`, or `-3905`/`-3912` timeouts.
- `wdutil info` → `Country Code : X2`, `Noise : 0 dBm`.
- airportd logs `Scan SUCCEEDED (... results=0 ...)`. **Careful:** the `liveCount=36/37` next to it is the number of *channels* scanned, not networks.
- The router logs no connection attempt at all.
- The same card in Windows (Broadcom driver 7.35.351.0) connects immediately.

## How to see the real cause

A failed join makes the driver write a CoreCapture dump:

```sh
ls -t /Library/Logs/CrashReporter/CoreCapture/
# e.g.  [...]=AuthFail{sts:5,rsn:0}   [...]=AssocFail{sts:5,rsn:0}   [...]=DeauthInd{sts:0,rsn:34}
D="/Library/Logs/CrashReporter/CoreCapture/<latest>/Data/com.apple.driver.AirPort.Brcm4360.0"
sudo head -60 "$D/StateSnapshots/Stats.txt"                  # tx/rx counters
sudo gzcat "$D"/DriverLogs/*.gz | grep -A8 "Power/Rate Dump"   # per-rate TX power, in 1/4 dB
sudo gzcat "$D"/DriverLogs/*.gz | grep "TxPower of Core"
```

What the dumps showed:

- `AuthFail{sts:5}`: Broadcom status 5 = **NO_ACK**. The AP never acknowledged our frames.
- `DeauthInd rsn:34`: the AP dropped us for too many unacknowledged frames.
- `The TxPower of Core No.0 is set to 1.0 dB` right after the regulatory setup for `X2`.
- `Power/Rate Dump (in 1/4dB)`: every rate at **12** (= 3 dBm). A normal card is around 60–72 (15–18 dBm).
- The radio itself was alive: noise floor around −89 dBm, beacons received. Reception was fine and transmission was nearly silent. Active scans send probe requests that no AP hears, so the scan comes back empty.

## Fix

1. Download **[AirportBrcmFixup](https://github.com/acidanthera/AirportBrcmFixup/releases)** (tested 2.2.1 RELEASE). Put only `AirportBrcmFixup.kext` in `EFI/OC/Kexts`. The `*_Injector.kext` plugins are for non‑Apple cards and are not needed for `14e4:43a0`.
2. In `config.plist` → `Kernel` → `Add`, add it **right after Lilu**:

   ```xml
   <dict>
     <key>Arch</key><string>Any</string>
     <key>BundlePath</key><string>AirportBrcmFixup.kext</string>
     <key>Comment</key><string>Country code + full TX power for BCM94360</string>
     <key>Enabled</key><true/>
     <key>ExecutablePath</key><string>Contents/MacOS/AirportBrcmFixup</string>
     <key>MaxKernel</key><string></string>
     <key>MinKernel</key><string></string>
     <key>PlistPath</key><string>Contents/Info.plist</string>
   </dict>
   ```

3. Add to `boot-args`:

   ```
   brcmfx-country=#a -amfipassbeta
   ```

   (`-amfipassbeta` is the usual AMFIPass flag on newer releases. It did not change anything here on its own.)
4. Remove the stale saved network if it exists, then reboot and join again:

   ```sh
   sudo networksetup -removepreferredwirelessnetwork en1 "<your SSID>"
   ```

### Why `#a` and not your country

With `brcmfx-country=IT` (any real country code should behave the same) the country code changed and **scans started working**. But the power table stayed around 3 dBm, and the link dropped seconds after association (`DeauthInd rsn:34`, no DHCP lease). Only `#a` gave full power and a stable link.

`#a` is a "no regulatory restrictions" code. On 2.4 GHz with this card's own power limits it stays at normal power, but it also unlocks channels and limits that may not be allowed where you live. Use it knowingly.

## Result

Joined a WPA2 2.4 GHz network at once: RSSI −75 dBm, link rate 59–73 Mbit/s, about 11 Mbit/s of real throughput through a floor. It still works after reboots and after updating OpenCore to 1.0.8.

## Things that did **not** help (so you can skip them)

- Reseating the antenna connectors (they were fine).
- A full cold boot (power removed) after using the card in Windows.
- `-amfipassbeta` alone.
- Turning on Location Services (needed for SSIDs to show in apps, but scans were empty anyway).
- Re-running the OCLP root patch.

## Credits

[AirportBrcmFixup](https://github.com/acidanthera/AirportBrcmFixup) by acidanthera / lvs1974, whose README mentions the *"txchain match to enable full transmit power on BCM94360"*. [OpenCore Legacy Patcher](https://github.com/dortania/OpenCore-Legacy-Patcher) by Dortania for the Modern Wireless patch and kexts.

Found something different on your card? Open an issue with your card model, macOS/OCLP versions and the `Power/Rate Dump` lines.
