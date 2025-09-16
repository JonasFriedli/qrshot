# qrshot — Screenshot a region and copy QR content to clipboard

`qrshot` lets you **select a screen region**, automatically **decode any QR codes** found, and **copy the decoded text to your clipboard**. Works on **X11** and **Wayland**.

---

## 1) Dependencies

### Wayland (most modern GNOME/KDE sessions)
- `grim` (screenshot)
- `slurp` (region selector)
- `zbar-tools` (QR decoding via `zbarimg`)
- `wl-clipboard` (clipboard: `wl-copy`)

### X11 (XFCE / older setups)
- `maim` (preferred) *or* `scrot` (fallback)
- `zbar-tools` (QR decoding via `zbarimg`)
- `xclip` (clipboard)

Install (you can install both stacks harmlessly):
```bash
sudo apt update
sudo apt install -y grim slurp wl-clipboard zbar-tools maim scrot xclip
```

> **Note:** `zbar-tools` provides `zbarimg` which is the QR decoder used by `qrshot`.

---

## 2) Install the `qrshot` command

Run this once to install the script system-wide so it’s available after reboot:

```bash
sudo tee /usr/local/bin/qrshot >/dev/null <<'BASH'
#!/usr/bin/env bash
set -euo pipefail

# Temporary file for the capture
png="$(mktemp /tmp/qrshot.XXXXXX.png)"
cleanup() { [[ -f "$png" ]] && rm -f "$png"; }
trap cleanup EXIT

# Detect session (Wayland vs X11)
if [[ -n "${WAYLAND_DISPLAY-}" ]]; then
  # Wayland path: grim + slurp + wl-copy
  if ! command -v grim >/dev/null || ! command -v slurp >/dev/null; then
    echo "grim/slurp missing (Wayland). Install: sudo apt install grim slurp" >&2
    exit 1
  fi
  geom="$(slurp)"
  grim -g "$geom" "$png"
  CLIP_CMD=(wl-copy)
else
  # X11 path: prefer maim, fallback to scrot
  if command -v maim >/dev/null; then
    maim -s "$png"
  elif command -v scrot >/dev/null; then
    scrot -s "$png"
  else
    echo "maim/scrot missing (X11). Install: sudo apt install maim scrot" >&2
    exit 1
  fi
  CLIP_CMD=(xclip -selection clipboard)
fi

# Ensure zbarimg present
if ! command -v zbarimg >/dev/null; then
  echo "zbarimg missing. Install: sudo apt install zbar-tools" >&2
  exit 1
fi

# Decode QR(s) and copy to clipboard
decoded="$(zbarimg --raw "$png" || true)"
if [[ -z "$decoded" ]]; then
  echo "No QR codes found in the selected region."
  exit 2
fi

# Trim trailing newline(s)
decoded="${decoded%%$'\n'}"

# Copy to clipboard
if ! command -v "${CLIP_CMD[0]}" >/dev/null; then
  if [[ -n "${WAYLAND_DISPLAY-}" ]]; then
    echo "Clipboard tool not found. Install: sudo apt install wl-clipboard" >&2
  else
    echo "Clipboard tool not found. Install: sudo apt install xclip" >&2
  fi
  echo "Decoded text (not copied):"
  printf '%s\n' "$decoded"
  exit 3
fi

printf '%s' "$decoded" | "${CLIP_CMD[@]}"
echo "QR copied to clipboard:"
printf '%s\n' "$decoded"
BASH
sudo chmod +x /usr/local/bin/qrshot
```

This installs `/usr/local/bin/qrshot`, which is on most users’ default `PATH`.

---

## 3) Usage

Run:
```bash
qrshot
```
- Draw a rectangle over the QR code.
- The decoded content is printed and copied to your clipboard.

If no QR is found, a helpful message is shown and exit code `2` is returned.

---

## 4) Tips & Troubleshooting

- **Multiple QRs:** `zbarimg` will output multiple lines. All lines are copied.  
- **Low-quality image:** Zoom in before selecting, or try selecting a tighter region around the QR.  
- **Clipboard missing:** If you see a clipboard error, install the missing tool (`wl-clipboard` on Wayland or `xclip` on X11).  
- **Permission issues:** Ensure `/usr/local/bin` exists and is writable with `sudo`.  
- **PATH:** If `qrshot` is not found, ensure `/usr/local/bin` is in your `PATH` (log out/in or `hash -r`).

---

## 5) Uninstall

```bash
sudo rm -f /usr/local/bin/qrshot
```
