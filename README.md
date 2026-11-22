# Autoscroll for Wayland

Windows-style middle-click autoscroll for Ubuntu (Wayland, GNOME).

- Middle-click drops a sprite at the cursor and turns scroll mode on.
- Move the mouse away from the sprite. Scrolling starts in that direction and speeds up the farther you move.
- Middle-click again to turn it off.

Middle-click paste is disabled, because the middle button now toggles scroll mode.

## Table of Contents

- [1. Install and set permissions](#1-install-and-set-permissions)
- [2. Create the daemon](#2-create-the-daemon)
- [3. Install the sprite extension](#3-install-the-sprite-extension)
- [4. Log out and back in](#4-log-out-and-back-in)
- [5. Enable and run](#5-enable-and-run)
- [How to use](#how-to-use)
- [Troubleshooting](#troubleshooting)

## 1. Install and set permissions

```bash
sudo apt install python3-evdev
sudo usermod -aG input $USER
echo 'KERNEL=="uinput", GROUP="input", MODE="0660", OPTIONS+="static_node=uinput"' | sudo tee /etc/udev/rules.d/99-uinput.rules
sudo udevadm control --reload-rules && sudo udevadm trigger
```

- `apt install python3-evdev` installs the library that reads mouse events.
- `usermod -aG input $USER` adds you to the `input` group so the script can read the mouse. It takes effect after you log out and back in.
- The `echo ... | sudo tee` line writes a udev rule that lets the `input` group create the virtual mouse.
- `udevadm` reloads the rule so it applies without a reboot.

## 2. Create the daemon

```bash
nano ~/autoscroll.py
```

Paste:

```python
#!/usr/bin/env python3
import asyncio, json, os
from evdev import InputDevice, UInput, ecodes as e, list_devices

STATE_FILE = f"/run/user/{os.getuid()}/autoscroll.json"
DEADZONE = 10      # px from the anchor before scrolling starts
SPEED = 10.0       # hi-res wheel units/sec per px beyond deadzone (120 = one notch)
MAX_SPEED = 3000   # cap, about 25 notches/sec
TICK = 1 / 120

WHEEL_CODES = (e.REL_WHEEL, e.REL_WHEEL_HI_RES, e.REL_HWHEEL, e.REL_HWHEEL_HI_RES)

def find_mouse():
    for path in list_devices():
        dev = InputDevice(path)
        if "autoscroll" in dev.name.lower():
            continue
        caps = dev.capabilities()
        if (e.EV_REL in caps and e.REL_X in caps[e.EV_REL] and e.REL_Y in caps[e.EV_REL]
                and e.EV_KEY in caps and e.BTN_MIDDLE in caps[e.EV_KEY]):
            return dev
    raise SystemExit("No mouse with a middle button found")

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
    ui = UInput(
        {e.EV_KEY: [e.BTN_LEFT, e.BTN_RIGHT, e.BTN_MIDDLE, e.BTN_SIDE, e.BTN_EXTRA],
         e.EV_REL: [e.REL_X, e.REL_Y, e.REL_WHEEL, e.REL_WHEEL_HI_RES,
                    e.REL_HWHEEL, e.REL_HWHEEL_HI_RES]},
        name="autoscroll-virtual-mouse")
    mouse.grab()

    active = False
    off_x = off_y = 0.0
    acc_v = acc_h = 0.0

    def set_active(on):
        nonlocal active, off_x, off_y, acc_v, acc_h
        active = on
        off_x = off_y = 0.0
        acc_v = acc_h = 0.0
        write_state(on)

    async def reader():
        nonlocal off_x, off_y
        async for ev in mouse.async_read_loop():
            # Middle button toggles the mode. Its press and release never reach the desktop.
            if ev.type == e.EV_KEY and ev.code == e.BTN_MIDDLE:
                if ev.value == 1:
                    set_active(not active)
                continue
            # Any other button press while active ends the mode and passes through.
            if active and ev.type == e.EV_KEY and ev.value == 1:
                set_active(False)
            elif active and ev.type == e.EV_REL:
                if ev.code == e.REL_X:
                    off_x += ev.value
                elif ev.code == e.REL_Y:
                    off_y += ev.value
                elif ev.code in WHEEL_CODES:
                    continue  # swallow physical wheel while in scroll mode
            # Motion still reaches the desktop, so the cursor moves away from the sprite.
            ui.write_event(ev)

    async def ticker():
        nonlocal acc_v, acc_h
        while True:
            await asyncio.sleep(TICK)
            if not active:
                continue
            # Screen-down motion = scroll down (negative wheel). Right = scroll right (positive HWHEEL).
            acc_v += -rate(off_y) * TICK
            acc_h += rate(off_x) * TICK
            iv, ih = int(acc_v), int(acc_h)
            if iv or ih:
                acc_v -= iv
                acc_h -= ih
                if iv:
                    ui.write(e.EV_REL, e.REL_WHEEL_HI_RES, iv)
                if ih:
                    ui.write(e.EV_REL, e.REL_HWHEEL_HI_RES, ih)
                ui.syn()

    write_state(False)
    try:
        await asyncio.gather(reader(), ticker())
    finally:
        write_state(False)
        mouse.ungrab()
        ui.close()

if __name__ == "__main__":
    asyncio.run(main())
```

Save with `Ctrl+O`, `Enter`, then exit with `Ctrl+X`. The file is in your home folder, so plain `nano` is right.

## 3. Install the sprite extension

Create the folder (no `sudo`, so you own the files):

```bash
mkdir -p ~/.local/share/gnome-shell/extensions/autoscroll-sprite@local
```

Create the manifest. Use plain `nano` here, not `sudo nano`, so the file stays owned by you:

```bash
nano ~/.local/share/gnome-shell/extensions/autoscroll-sprite@local/metadata.json
```

Paste:

```json
{
  "uuid": "autoscroll-sprite@local",
  "name": "Autoscroll Sprite",
  "description": "Draws the autoscroll anchor sprite",
  "shell-version": ["46", "47", "48", "49", "50"]
}
```

Create the extension code:

```bash
nano ~/.local/share/gnome-shell/extensions/autoscroll-sprite@local/extension.js
```

Paste:

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

Save with `Ctrl+O`, `Enter`, then exit with `Ctrl+X`.

## 4. Log out and back in

GNOME only scans for extensions at login, so this step is required before the extension will show up.

## 5. Enable and run

```bash
gnome-extensions enable autoscroll-sprite@local
python3 ~/autoscroll.py
```

Leave the terminal open while you use it. Press `Ctrl+C` in that terminal to quit the script.

## How to use

The script has to be running for autoscroll to work. Start it with `python3 ~/autoscroll.py` in a terminal. It does not start on its own at login yet.

1. Put the pointer over the content you want to scroll: a web page, a PDF, a document, or a code file.
2. Middle-click once. A small ring appears under the cursor. That spot is the anchor.
3. Move the mouse down to scroll down, or up to scroll up. The farther you move from the ring, the faster it scrolls.
4. Move back toward the ring to slow down or stop. Moving left or right scrolls horizontally.
5. Middle-click again to stop. Clicking any other mouse button also stops it.

Notes:

- The cursor still moves while scrolling, so you can aim the pointer at the part of the page you want.
- While scroll mode is on, your physical scroll wheel is ignored.
- Middle-click paste no longer works, because the middle button toggles scroll mode.
- To change the feel, edit `SPEED` or `MAX_SPEED` at the top of `~/autoscroll.py`, save, then press `Ctrl+C` and run the script again.

If the mouse ever stops responding, switch to a text console with `Ctrl+Alt+F3`, then press `Ctrl+C` in the terminal where the script is running, or run `pkill -f autoscroll.py`. Stopping the script releases the mouse.

## Troubleshooting

- **No sprite:** run `gnome-extensions info autoscroll-sprite@local` and check for `State: ACTIVE`. If the state is `ERROR`, check `journalctl --user -b | grep -i autoscroll`.
- **"No mouse with a middle button found":** list your input devices with `python3 -c "from evdev import list_devices, InputDevice; [print(p, InputDevice(p).name) for p in list_devices()]"`, then send the output so the script can be pinned to your device.
- **Permission errors:** confirm `groups` lists `input`. If it doesn't, log out and back in.
- **Too slow or too fast:** change `SPEED` (and `MAX_SPEED` for the top speed) at the top of `~/autoscroll.py`, then restart the script.
