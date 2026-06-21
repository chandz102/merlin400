# Making `merlin400-system` the default boot software

A short guide for switching a Merlin400's Raspberry Pi from the **original
"drizzle" software** to the aftermarket **`merlin400-system`** software, so the
aftermarket one starts automatically every time the machine powers on.

> You only need to do this **once**. After that it sticks across reboots.

---

## Before you start

You need:
- The `merlin400-system` software already installed on the Pi (there should be a
  `merlin400-system` systemd service — check with the first command below).
- SSH access to the Pi, e.g. `ssh pi@<pi-ip-address>` (default user is usually `pi`).
- The ability to run `sudo` (admin) commands.

Check the service exists:
```bash
ssh pi@<PI_IP_ADDRESS> "systemctl list-unit-files | grep -E 'drizzle|merlin400'"
```
You should see both `drizzle` and `merlin400-system`. If you don't see
`merlin400-system`, it isn't installed yet — install it first.

---

## Step 1 — Stop the original software (right now, this session)

```bash
ssh pi@<PI_IP_ADDRESS> "sudo systemctl stop drizzle"
ssh pi@<PI_IP_ADDRESS> "sudo pkill -f python"
```
- `stop drizzle` shuts down the original service.
- `pkill -f python` clears any leftover Python process that was still holding the
  hardware (the controller is a Python program).

## Step 2 — Start the aftermarket software (right now)

```bash
sudo systemctl start merlin400-system
```

Check it came up cleanly:
```bash
ssh pi@<PI_IP_ADDRESS> "sudo systemctl status merlin400-system"
```
Look for **`active (running)`**. Press `q` to exit the status view.

## Step 3 — Make the change permanent (survives reboots)

```bash
ssh pi@<PI_IP_ADDRESS> "sudo systemctl disable drizzle"           # stop the old one auto-starting
ssh pi@<PI_IP_ADDRESS> "sudo systemctl enable merlin400-system"   # make the new one auto-start
```
- `disable` / `enable` only affect what happens **at boot** — they don't start or
  stop anything right now (Steps 1–2 already did that).

---

## Verify it worked

1. Confirm boot settings:
   ```bash
   ssh pi@<PI_IP_ADDRESS> "systemctl is-enabled drizzle"            # should say:  disabled
   ssh pi@<PI_IP_ADDRESS> "systemctl is-enabled merlin400-system"   # should say:  enabled
   ```

2. The real test — **reboot and check it comes back on its own**:
   ```bash
   ssh pi@<PI_IP_ADDRESS> "sudo reboot"
   ```
   Wait ~30–60 seconds, reconnect, then:
   ```bash
   ssh pi@<PI_IP_ADDRESS> "systemctl is-active merlin400-system"    # should say:  active
   ssh pi@<PI_IP_ADDRESS> "curl -s http://localhost/api/status"     # should return JSON (machine status)
   ```
   If the API returns status and the machine is responsive, you're done. ✅

---

## How to undo it (revert to the original software)

If you ever want the original `drizzle` software back as default:
```bash
ssh pi@<PI_IP_ADDRESS> "sudo systemctl stop merlin400-system"
ssh pi@<PI_IP_ADDRESS> "sudo systemctl disable merlin400-system"
ssh pi@<PI_IP_ADDRESS> "sudo systemctl enable drizzle"
ssh pi@<PI_IP_ADDRESS> "sudo systemctl start drizzle"
```

---

## Troubleshooting

| Problem | What to do |
|---------|-----------|
| `merlin400-system` won't start | View the logs: `ssh pi@<PI_IP_ADDRESS> "sudo journalctl -u merlin400-system -n 50 --no-pager"` |
| "Address already in use" / port error | The old software or a stray Python process is still running. Run the stop and pkill commands above, then start again. |
| Can't reach the web page after reboot | Confirm it's enabled and active with the commands above; check logs as above. |
| `sudo: command not found` / permission denied | Make sure you're logged in as a user with admin rights (usually `pi`). |

---

## What each command actually does (plain English)

- **`systemctl stop X`** — turn service X off *now*.
- **`systemctl start X`** — turn service X on *now*.
- **`systemctl disable X`** — don't start X automatically at boot.
- **`systemctl enable X`** — start X automatically at boot.
- **`systemctl status X`** — show whether X is running and recent log lines.
- **`systemctl is-enabled X` / `is-active X`** — quick yes/no on boot-setting / running.
