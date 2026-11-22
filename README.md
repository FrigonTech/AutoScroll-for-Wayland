# Autoscroll for Wayland

Windows-style middle-click autoscroll that works system-wide on Ubuntu with a Wayland session.

- **Middle-click** drops a sprite at the cursor and turns scroll mode on.
- **Move the mouse** away from the sprite. Scrolling starts in that direction and speeds up the farther you move.
- **Middle-click again**, or click any other button, to turn scroll mode off.

Trade-off: middle-click paste is disabled, because the middle button now toggles scroll mode.

---

## How it works

There are two parts:

1. **`autoscroll.py`** (the daemon) reads your physical mouse through `evdev`. It creates a virtual mouse through `uinput` and passes everything through to it, except the middle button and the wheel while scroll mode is on. It also writes a small state file that says whether scroll mode is on.
2. **A GNOME Shell extension** draws the sprite at the cursor when the state file says scroll mode is on.

Wayland does not let normal apps draw at a fixed screen position, so the sprite has to come from the compositor, which is GNOME Shell on Ubuntu. The daemon works below the display server, so it works in every app.

## Requirements

- Ubuntu 26.04 with the default GNOME Wayland session
- GNOME Shell 46 to 50. Check with `gnome-shell --version`.
- A user account with `sudo`

---

## Step 1: Install the Python library

```bash
sudo apt update
sudo apt install -y python3-evdev
```

- `sudo apt update` refreshes the list of available packages.
- `sudo apt install -y python3-evdev` installs the library the daemon uses to read mouse events. `-y` answers yes to the prompt.

## Step 2: Give your user access to mouse and virtual-mouse devices

Add your user to the `input` group so the daemon can read the mouse:

```bash
sudo usermod -aG input $USER
```

- `usermod -aG input $USER` adds your user (`$USER` expands to your username) to the `input` group. `-a` means append, so existing groups are kept. This only takes effect after you log out and back in, which you'll do in Step 7.

Create a udev rule so the `input` group can create virtual devices:

```bash
sudo nano /etc/udev/rules.d/99-uinput.rules
```

Paste this line into nano:

```
KERNEL=="uinput", GROUP="input", MODE="0660", OPTIONS+="static_node=uinput"
```

Save and exit: press `Ctrl+O`, then `Enter`, then `Ctrl+X`.

- `sudo nano <file>` opens a file in the nano editor as root. Files under `/etc` need root, so `sudo` is required here.
- The rule says: when the kernel creates the `uinput` device, give it group `input` and permissions `0660`, so group members can read and write it.

Make sure the `uinput` kernel module loads at boot:

```bash
sudo nano /etc/modules-load.d/uinput.conf
```

Paste this single word into nano:

```
uinput
```

Save and exit with `Ctrl+O`, `Enter`, `Ctrl+X`.

Load the module now and apply the udev rule:

```bash
sudo modprobe uinput
sudo udevadm control --reload-rules
sudo udevadm trigger
```

- `modprobe uinput` loads the virtual input driver without rebooting.
- `udevadm control --reload-rules` makes udev read the new rule.
- `udevadm trigger` applies the rules to devices that already exist.

> **Note on `sudo` and home-folder files:** Use `sudo nano` only for system files such as the ones in `/etc`. For files in your home folder, use plain `nano`. Running `sudo nano` on a home file creates a root-owned file that you can't edit later without `sudo`.

## Step 3: Create the folders

```bash
mkdir -p ~/.local/bin
mkdir -p ~/.local/share/gnome-shell/extensions/autoscroll-sprite@local
mkdir -p ~/.config/systemd/user
```

- `mkdir -p` creates a folder and any missing parent folders. It does nothing if the folder already exists.
- `~/.local/bin` holds the daemon script.
- `~/.local/share/gnome-shell/extensions/autoscroll-sprite@local` is where GNOME looks for extensions. The folder name must match the extension's UUID exactly.
- `~/.config/systemd/user` holds the service file that runs the daemon in the background.

## Step 4: Create the daemon

```bash
nano ~/.local/bin/autoscroll.py
```

Paste this code:

```python
#!/usr/bin/env python3
"""Windows-style middle-click autoscroll for Wayland, using evdev and uinput.

Middle click toggles scroll mode and anchors it at the cursor position.
Moving the mouse away from the anchor scrolls in that direction, faster the
farther you move. Middle click again (or any other button) turns it off.
"""
import asyncio
import json
import os
import signal

from evdev import InputDevice, UInput, ecodes as e, list_devices

STATE_FILE = f"/run/user/{os.getuid()}/autoscroll.json"
WANT_NAME = os.environ.get("AUTOSCROLL_DEVICE", "").lower()  # optional: part of your mouse's name
DEADZONE = 10       # px from the anchor before scrolling starts
SPEED = 10.0        # hi-res wheel units per second, per px beyond the deadzone (120 = one notch)
MAX_SPEED = 3000    # cap in hi-res units per second (3000 is about 25 notches/sec)
TICK = 1 / 120      # seconds between scroll updates

WHEEL_CODES = (e.REL_WHEEL, e.REL_WHEEL_HI_RES, e.REL_HWHEEL, e.REL_HWHEEL_HI_RES)


def find_mouse():
    for path in list_devices():
        dev = InputDevice(path)
        name = dev.name.lower()
        if "autoscroll" in name or (WANT_NAME and WANT_NAME not in name):
            continue
        caps = dev.capabilities()
        if (e.EV_REL in caps and e.REL_X in caps[e.EV_REL] and e.REL_Y in caps[e.EV_REL]
                and e.EV_KEY in caps and e.BTN_MIDDLE in caps[e.EV_KEY]):
            return dev
    raise SystemExit("No matching mouse with a middle button found")


def write_state(active):
    tmp = STATE_FILE + ".tmp"
    with open(tmp, "w") as f:
        json.dump({"active": active}, f)
    os.replace(tmp, STATE_FILE)


def rate(d):
    """Scroll rate in hi-res units per second for an offset d from the anchor."""
    over = abs(d) - DEADZONE
    if over <= 0:
        return 0.0
    r = min(over * SPEED, MAX_SPEED)
    return r if d > 0 else -r


async def main():
    mouse = find_mouse()
    caps = mouse.capabilities()
    rels = set(caps.get(e.EV_REL, [])) | set(WHEEL_CODES)
    ui = UInput(
        {e.EV_KEY: list(caps.get(e.EV_KEY, [])), e.EV_REL: sorted(rels)},
        name="autoscroll-virtual-mouse",
    )
    mouse.grab()

    active = False
    off_x = off_y = 0.0      # total movement since the anchor
    acc_v = acc_h = 0.0      # fractional hi-res units carried between ticks
    legacy_v = legacy_h = 0  # hi-res units not yet sent as whole REL_WHEEL notches

    def set_active(on):
        nonlocal active, off_x, off_y, acc_v, acc_h
        active = on
        off_x = off_y = 0.0
        acc_v = acc_h = 0.0
        write_state(on)

    async def reader():
        nonlocal off_x, off_y
        async for ev in mouse.async_read_loop():
            if ev.type == e.EV_KEY and ev.code == e.BTN_MIDDLE:
                if ev.value == 1:
                    set_active(not active)
                continue                     # middle button never reaches the desktop
            if active and ev.type == e.EV_KEY and ev.value == 1:
                set_active(False)            # any other button press ends scroll mode
            elif active and ev.type == e.EV_REL:
                if ev.code == e.REL_X:
                    off_x += ev.value
                elif ev.code == e.REL_Y:
                    off_y += ev.value
                elif ev.code in WHEEL_CODES:
                    continue                 # swallow physical wheel while in scroll mode
            ui.write_event(ev)               # motion still moves the cursor

    async def ticker():
        nonlocal acc_v, acc_h, legacy_v, legacy_h
        while True:
            await asyncio.sleep(TICK)
            if not active:
                continue
            acc_v += -rate(off_y) * TICK     # moving down scrolls down (negative wheel)
            acc_h += rate(off_x) * TICK      # moving right scrolls right (positive hwheel)
            iv, ih = int(acc_v), int(acc_h)
            if not (iv or ih):
                continue
            acc_v -= iv
            acc_h -= ih
            if iv:
                ui.write(e.EV_REL, e.REL_WHEEL_HI_RES, iv)
                legacy_v += iv
                notches = int(legacy_v / 120)
                if notches:
                    legacy_v -= notches * 120
                    ui.write(e.EV_REL, e.REL_WHEEL, notches)
            if ih:
                ui.write(e.EV_REL, e.REL_HWHEEL_HI_RES, ih)
                legacy_h += ih
                notches = int(legacy_h / 120)
                if notches:
                    legacy_h -= notches * 120
                    ui.write(e.EV_REL, e.REL_HWHEEL, notches)
            ui.syn()

    loop = asyncio.get_running_loop()
    tasks = [asyncio.create_task(reader()), asyncio.create_task(ticker())]
    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(sig, lambda: [t.cancel() for t in tasks])

    write_state(False)
    try:
        await asyncio.gather(*tasks, return_exceptions=True)
    finally:
        write_state(False)
        mouse.ungrab()
        ui.close()


if __name__ == "__main__":
    asyncio.run(main())
```

Save and exit with `Ctrl+O`, `Enter`, `Ctrl+X`.

- `nano <file>` opens the file for editing, creating it if it doesn't exist.
- Paste with `Ctrl+Shift+V` in most Ubuntu terminals.
- The file lives in your home folder, so plain `nano` is correct here.

## Step 5: Create the sprite extension

Create the extension manifest:

```bash
nano ~/.local/share/gnome-shell/extensions/autoscroll-sprite@local/metadata.json
```

Paste this code:

```json
{
  "uuid": "autoscroll-sprite@local",
  "name": "Autoscroll Sprite",
  "description": "Draws the autoscroll anchor sprite",
  "shell-version": ["46", "47", "48", "49", "50"]
}
```

Save and exit with `Ctrl+O`, `Enter`, `Ctrl+X`.

- `uuid` must match the folder name.
- `shell-version` lists the GNOME versions the extension supports. GNOME refuses to load an extension whose list doesn't include your major version. If `gnome-shell --version` shows a version that isn't in the list, add it.

Create the extension code:

```bash
nano ~/.local/share/gnome-shell/extensions/autoscroll-sprite@local/extension.js
```

Paste this code:

```javascript
import St from 'gi://St';
import Gio from 'gi://Gio';
import GLib from 'gi://GLib';
import {Extension} from 'resource:///org/gnome/shell/extensions/extension.js';

const SIZE = 28;

export default class AutoscrollSprite extends Extension {
    enable() {
        this._active = false;
        this._path = `${GLib.get_user_runtime_dir()}/autoscroll.json`;

        this._sprite = new St.Widget({
            width: SIZE,
            height: SIZE,
            visible: false,
            reactive: false,
            style: `border-radius: ${SIZE}px;
                    border: 2px solid rgba(255,255,255,0.9);
                    background-color: rgba(0,0,0,0.45);`,
        });
        this._dot = new St.Widget({
            width: 6,
            height: 6,
            x: (SIZE - 6) / 2,
            y: (SIZE - 6) / 2,
            style: 'border-radius: 3px; background-color: white;',
        });
        this._sprite.add_child(this._dot);
        global.stage.add_child(this._sprite);

        this._timer = GLib.timeout_add(GLib.PRIORITY_DEFAULT, 30, () => {
            this._poll();
            return GLib.SOURCE_CONTINUE;
        });
    }

    _poll() {
        let state;
        try {
            const [ok, contents] = Gio.File.new_for_path(this._path).load_contents(null);
            if (!ok)
                return;
            state = JSON.parse(new TextDecoder().decode(contents));
        } catch (e) {
            return;
        }

        if (state.active === this._active)
            return;
        this._active = state.active;

        if (this._active) {
            const [x, y] = global.get_pointer();
            this._sprite.set_position(x - SIZE / 2, y - SIZE / 2);
            this._sprite.show();
        } else {
            this._sprite.hide();
        }
    }

    disable() {
        if (this._timer) {
            GLib.source_remove(this._timer);
            this._timer = null;
        }
        this._sprite?.destroy();
        this._sprite = null;
        this._dot = null;
    }
}
```

Save and exit with `Ctrl+O`, `Enter`, `Ctrl+X`.

- GNOME Shell runs this file inside the desktop, so it needs the `gi://` imports shown here.
- The extension checks a small state file every 30 ms. The daemon writes that file, so the sprite follows the daemon's toggle.

Check that the JSON is valid:

```bash
python3 -m json.tool ~/.local/share/gnome-shell/extensions/autoscroll-sprite@local/metadata.json
```

It should print the manifest back with no error. If it reports an error, reopen the file with `nano` and look for a stray character.

## Step 6: Create the background service

```bash
nano ~/.config/systemd/user/autoscroll.service
```

Paste this code:

```ini
[Unit]
Description=Windows-style autoscroll for Wayland

[Service]
ExecStart=/usr/bin/python3 %h/.local/bin/autoscroll.py
Restart=on-failure
RestartSec=2
# Uncomment and set this if the script picks the wrong device.
# The value is part of your mouse's name, which you can find in Troubleshooting.
# Environment=AUTOSCROLL_DEVICE=Logitech

[Install]
WantedBy=default.target
```

Save and exit with `Ctrl+O`, `Enter`, `Ctrl+X`.

- A user service is managed by `systemctl --user`, so it runs in your session without `sudo`.
- `%h` means your home folder.
- `Restart=on-failure` starts the daemon again if it crashes.

## Step 7: Log out and back in

Log out from the GNOME menu, then log back in.

This does two things:

- It applies the `input` group membership from Step 2.
- It makes GNOME scan `~/.local/share/gnome-shell/extensions/`. GNOME only scans for new extensions at login, so a folder created during the current session won't appear until you log in again.

## Step 8: Enable the extension and start the daemon

```bash
gnome-extensions enable autoscroll-sprite@local
gnome-extensions info autoscroll-sprite@local
systemctl --user daemon-reload
systemctl --user enable --now autoscroll.service
systemctl --user status autoscroll.service
```

- `gnome-extensions enable` turns the extension on.
- `gnome-extensions info` shows its state. Look for `Enabled: Yes` and `State: ACTIVE`.
- `systemctl --user daemon-reload` makes systemd read the new service file.
- `systemctl --user enable --now autoscroll.service` starts the daemon now and at every login.
- `systemctl --user status` should show `active (running)`.

## Step 9: Test it

1. Open a web page or a long text file.
2. Middle-click. A small ring should appear under the cursor.
3. Move the mouse down. The page should start scrolling down and speed up as you move farther away.
4. Move back toward the ring to slow down or stop. Move up to scroll up.
5. Middle-click again to end scroll mode.

---

## Troubleshooting

**The sprite doesn't appear but scrolling works**

- Run `gnome-extensions info autoscroll-sprite@local` and confirm `State: ACTIVE`. If it says `ERROR`, run `journalctl --user -b | grep -i autoscroll` and look for the cause.
- Confirm `gnome-shell --version` matches a version in `metadata.json`.
- If you created the folder after logging in, log out and back in once more.

**The service says "No matching mouse with a middle button found"**

List the input devices:

```bash
python3 -c "from evdev import list_devices, InputDevice; [print(p, InputDevice(p).name) for p in list_devices()]"
```

Find your mouse in the names. Then uncomment the `Environment=` line in `autoscroll.service`, set it to a unique part of the name, and restart:

```bash
nano ~/.config/systemd/user/autoscroll.service
systemctl --user daemon-reload
systemctl --user restart autoscroll.service
```

If the command fails with a permission error, your `input` group membership hasn't applied yet. Log out and back in.

**"Permission denied" on `/dev/uinput` or `/dev/input`**

- Run `groups` and confirm `input` is listed. If not, repeat Step 7.
- Run `ls -l /dev/uinput` and confirm the group is `input`. If not, repeat the udev steps in Step 2.

**Scrolling is too slow or too fast**

Edit the numbers at the top of `~/.local/bin/autoscroll.py`:

- `DEADZONE` is how far you must move before scrolling starts.
- `SPEED` controls how quickly scroll speed rises with distance.
- `MAX_SPEED` caps the top speed.

Then restart the service:

```bash
nano ~/.local/bin/autoscroll.py
systemctl --user restart autoscroll.service
```

**Logs**

```bash
journalctl --user -u autoscroll.service -n 50
```

**Emergency: the mouse seems stuck**

Switch to a text console with `Ctrl+Alt+F3`, log in, and run:

```bash
systemctl --user stop autoscroll.service
```

Stopping the daemon releases the mouse. Press `Ctrl+Alt+F2` (or `F1`/`F3` as needed) to return to the desktop.

---

## Uninstall

```bash
systemctl --user disable --now autoscroll.service
gnome-extensions disable autoscroll-sprite@local
rm -r ~/.local/share/gnome-shell/extensions/autoscroll-sprite@local
rm ~/.local/bin/autoscroll.py ~/.config/systemd/user/autoscroll.service
systemctl --user daemon-reload
sudo rm /etc/udev/rules.d/99-uinput.rules /etc/modules-load.d/uinput.conf
sudo gpasswd -d $USER input
```

- `systemctl --user disable --now` stops the daemon and removes it from autostart.
- `rm -r` deletes the extension folder. The plain `rm` commands delete the daemon and service file.
- `sudo rm` removes the system files you created with `sudo nano`.
- `sudo gpasswd -d $USER input` removes you from the `input` group. Log out and back in afterward.
