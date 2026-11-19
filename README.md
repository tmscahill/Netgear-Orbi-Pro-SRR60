# Orbi Pro SRR60 Attached Devices overrides

Tested on a **NETGEAR Orbi Pro SRR60** running **firmware V2.7.2.102**.

**Use at your own risk.** These commands run as root, and `d2 -c` writes to the Orbi's persistent configuration. Changing the wrong record or field could royally toast your Orbi. Back up your configuration, verify the target MAC before writing, and have a recovery plan.

## What happened

My wired Raspberry Pi showed up in Attached Devices with the name `1` and model `Chromebook`. The web UI submitted the requested name `raspberrypi` and model `pihole`, but the live list kept showing the old values. Writing the custom fields through the router's DAL utility saved both values to flash and changed the rebuilt list. Reboot persistence was untested.

This walkthrough assumes existing root shell access. The MAC, slot, and process ID are examples from the tested router.

## 1. Identify the generated list

The web UI requests `DEV_device_first_info.htm` on initial load and `DEV_device_info.htm` on refresh. A trace of the web backend showed it opening `/tmp/netscan/attach_device` for reading. Editing that file changed the display until the next scan rebuilt it.

Set the MAC to your target device:

```sh
mac='B8:27:EB:6A:3F:19'
grep -i -A 8 "$mac" /tmp/netscan/attach_device
```

## 2. Locate the persistent record and verify its slot

The process listing showed `/usr/bin/d2d /tmp/dal/d2d`. On the tested router, `/tmp/dal` was mounted from `/dev/mmcblk0p31`, separately from the RAM-backed `/tmp`.

```sh
df /tmp /tmp/dal/d2d
grep -ilF "$mac" /tmp/dal/d2d/DeviceIdList.*.json
```

My device matched `DeviceIdList.07A.json`. The hexadecimal suffix `07A` corresponds to decimal slot `122`. Substitute your matching filename and slot. Confirm that the live MAC is the intended device, and record the current custom values before changing them.

```sh
record=/tmp/dal/d2d/DeviceIdList.07A.json
slot=122
key="deviceidlist[$slot]"
/usr/bin/d2 -s "$key.DeviceMac"
/usr/bin/d2 -s "$key.DisplayNameCustom"
/usr/bin/d2 -s "$key.DisplayModelCustom"
md5sum "$record"
```

## 3. Trace the web save if you want to diagnose the failure

In the browser's Network panel, the Apply POST included `submit_flag=orbi_devname`, `devname_changed=1`, `orbi_name=raspberrypi`, and `hid_orbi_model=pihole`. The response was a wait page with a fallback redirect. The persistent record's checksum was unchanged after the save.

Find the current `lighttpd` PID before tracing. The example uses PID 10790.

```sh
ps w | grep '[l]ighttpd'
strace -f -tt -s 256 -e trace=execve -p 10790 -o /tmp/orbi-apply.trace
```

Click **Apply**, wait 30 seconds, and stop the trace with Ctrl+C. This prints the five highest DAL slots queried.

```sh
sed -n 's/.*deviceidlist\[\([0-9][0-9]*\)\]\.DeviceMac".*/\1/p' /tmp/orbi-apply.trace |
  sort -nu | tail -5
```

Two captures ended at slot 54, whose MAC was all zeros. Slot 55 contained a real device, and the Pi was reachable at slot 122. The likely explanation is that the lookup stops at the first empty slot. The trace established where the observed lookups ended. The firmware's loop logic remains unverified.

## 4. Apply the custom name and model

With `mac` and `key` set as above in the same shell, verify the slot's MAC again immediately before writing. If the second command fails, the first change may already have been applied. Check both fields afterward.

```sh
if [ "$(/usr/bin/d2 -s "$key.DeviceMac" | tr 'a-f' 'A-F')" = "$(printf '%s' "$mac" | tr 'a-f' 'A-F')" ] &&
   [ -n "$mac" ]; then
  /usr/bin/d2 -c "$key.DisplayNameCustom" 'raspberrypi' &&
  /usr/bin/d2 -c "$key.DisplayModelCustom" 'pihole'
else
  echo 'MAC check failed. No changes made.'
fi
```

## 5. Verify the live, flash-backed, and generated records

Reload Attached Devices and let it refresh. Check the live DAL values, the persistent JSON, and the generated list.

```sh
/usr/bin/d2 -s "$key.DisplayNameCustom"
/usr/bin/d2 -s "$key.DisplayModelCustom"
tr ',' '\n' < "$record" | grep -E '"Display(Name|Model)Custom"'
grep -i -A 8 "$mac" /tmp/netscan/attach_device
```

On my router, all three showed the name `raspberrypi` and model `pihole` for the target MAC.

## AI assistance

I used OpenAI Codex (GPT-6) to help interpret the router's logs and traces and identify the DAL commands. I ran the commands on my Orbi Pro SRR60 and verified the results myself.
