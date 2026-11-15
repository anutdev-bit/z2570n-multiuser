# Android 16 — switching users when the vendor removed the Users screen

**Tested on:** nubia Neo 5GT (`Z2570N`), MyOS 16.0.3 TH `MyOS16.0.3_Z2570N_TH_V` (`20261025.121204`)

After a firmware update the **"Users" entry is gone from Settings**, `pm create-user` fails with
`code 6`, and an existing secondary user keeps holding tens of gigabytes with no UI way to delete it.

You cannot create users any more — but you can still **switch** between the users that already exist.
No root needed. Just adb and one command.

---

## Step 1 — connect adb

**Option A (USB):** plug the cable, unlock the screen, accept the "Allow USB debugging?" prompt.

**Option B (Wi-Fi, no cable):** Settings → Developer options → **Wireless debugging** → on, then on a
host machine:

```sh
adb mdns services | grep _adb-tls-connect      # -> <phone-ip>:<port>
adb connect <phone-ip>:<port>
```

**Option C (phone has no Wi-Fi to pair with):** Settings → Developer options → Wireless debugging →
**Pair device with pairing code**, then `adb pair <phone-ip>:<pair-port>` and enter the code.

Check it worked:

```sh
adb devices          # your device must show as "device", not "unauthorized"
```

## Step 2 — the commands

```sh
adb shell pm list users            # every user that still exists
adb shell am get-current-user      # 0 = primary, 10 = secondary, ...
adb shell am switch-user 0         # switch to the primary user
adb shell am switch-user 10        # switch to the secondary user
```

That's the whole workaround. `am switch-user` only needs the `shell` uid, which is what `adb shell`
already gives you.

**Expect adb to drop mid-command.** The phone's USB gadget resets during a user switch (it
re-enumerates as a MIDI device), so the connection dies right after `am switch-user` — that is the
command working, not failing. Replug the cable, or reconnect with `adb connect`, and continue.

## Step 3 — verify

```sh
adb shell am get-current-user
adb shell dumpsys user | head -30      # who exists, who was last logged in
```

## Helper script

`scripts/nubia` wraps all of the above (finds the device over USB or mDNS, then runs the command):

```sh
./scripts/nubia status            # device, current user
./scripts/nubia users             # pm list users
./scripts/nubia switch owner      # am switch-user 0
./scripts/nubia switch 10         # am switch-user 10
./scripts/nubia usb mtp           # set USB mode to file transfer
./scripts/nubia sh 'pm get-max-users'
```

Any Android device works; nothing here is model specific.

## Why the feature is gone (2 lines)

`framework-res.apk` still declares `config_multiuserMaximumUsers = 4`, but the vendor ships
**static, non-mutable RRO overlays** that cap it to 1. `dumpsys user` states it plainly:

```
Max users: 1 (built-in value: 4) (limit reached: true)
```

Not fixable without root: the overlays cannot be disabled, `setprop fw.max_users 4` is blocked by
SELinux, and the bootloader is locked. Check your own device the same way:

```sh
adb shell pm get-max-users
adb shell dumpsys user | grep -i 'max users'
adb shell cmd overlay list android | grep -i frameworkres
```

## Before you switch away from the primary user

These are the parts that bite people, all verified on this device:

- **A secondary user cannot start adb/Shizuku-class helpers.** Wireless debugging and its pairing UI
  are only reachable from the primary user.
- **Android boots into the last-used user.** A reboot, power-off, or dead battery while in a
  secondary user can leave you stuck until you have a PC again.
- **Never touch the overflow (⋮) menu** on the guest-mode Settings screen — it contains
  "Delete User from this device", which permanently wipes that user's data.
- Settings no longer offers any on-device escape ("Exit guest mode" is a no-op for a non-guest user).

Recovery: reconnect adb and run `adb shell am switch-user 0`.

## License

MIT.
