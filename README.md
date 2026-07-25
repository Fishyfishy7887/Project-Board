# ESP32-S3 BAT LIGHT wall switch

This project contains a single-purpose ESPHome configuration for the permanently
wired Lonely Binary ESP32-S3 Gold Edition and ST7796/FT63X6 4-inch touchscreen.
It keeps the SPI, I2C, display, touch transform, PSRAM, and ESP-IDF settings from
the passed hardware test described in the handoff. The UI redraws a dark
background, a bordered button, and its label; a press changes only the border
style before calling Home Assistant's `light.toggle` action for
`light.bat_light`.

## One-pass inspect, validate, compile, and upload

Run this on **caper-brains**, where the passed test, ESPHome virtual environment,
and USB device exist:

```bash
cd "$HOME/wall-switch-test" || exit 1

# Confirm the known-good initialization and compare it with the clean file.
sed -n '/^spi:/,/^[^[:space:]]/p; /^i2c:/,/^[^[:space:]]/p; /^display:/,/^[^[:space:]]/p; /^touchscreen:/,/^[^[:space:]]/p' \
  screen-touch-test-PASSED.yaml

diff -u \
  <(sed -n '/^spi:/,/^[^[:space:]]/p; /^i2c:/,/^[^[:space:]]/p; /^display:/,/^[^[:space:]]/p; /^touchscreen:/,/^[^[:space:]]/p' screen-touch-test-PASSED.yaml) \
  <(sed -n '/^spi:/,/^[^[:space:]]/p; /^i2c:/,/^[^[:space:]]/p; /^display:/,/^[^[:space:]]/p; /^touchscreen:/,/^[^[:space:]]/p' bat-light-wall-switch.yaml) || true

# Check only the secret keys and nonempty values; this never prints credentials.
python3 - <<'PY'
from pathlib import Path
import re
text = Path("secrets.yaml").read_text()
for key in ("wifi_ssid", "wifi_password", "fallback_ap_password"):
    match = re.search(rf"(?m)^\s*{key}:\s*(['\"])(.+)\1\s*$", text)
    if not match or not match.group(2).strip():
        raise SystemExit(f"secrets.yaml: missing or empty {key}")
print("secrets.yaml: required values are present (values hidden)")
PY

# Materialize the configured local font from caper-brains' installed fonts.
mkdir -p fonts
FONT_FILE="$(fc-match -f '%{file}\n' 'DejaVu Sans:style=Bold' | head -n 1)"
test -f "$FONT_FILE" || {
  echo "DejaVu Sans Bold was not found on caper-brains."
  exit 1
}
cp -f "$FONT_FILE" fonts/DejaVuSans-Bold.ttf

.venv/bin/esphome config bat-light-wall-switch.yaml && \
.venv/bin/esphome compile bat-light-wall-switch.yaml && \
sudo chmod 666 /dev/ttyUSB0 && \
.venv/bin/esphome upload bat-light-wall-switch.yaml --device /dev/ttyUSB0
```

If `secrets.yaml` does not yet have `fallback_ap_password`, add a unique value of
at least eight characters. If either Wi-Fi value is wrong, edit `secrets.yaml`
directly; do not paste credentials into logs or issue history.

## Home Assistant setup

1. Wait for `bat-light-wall-switch` to obtain an address from the router.
2. In Home Assistant, add the ESPHome integration. If discovery/mDNS does not
   work, enter the device's **numeric IP address**.
3. When prompted, enable **Allow the device to perform Home Assistant actions**.
   The touchscreen action will not be accepted without that permission.
4. Press the outlined BAT LIGHT button once. Its cyan border briefly becomes a
   thicker amber border, and `light.bat_light` should toggle.

The fallback access point is named `BAT Light Setup`. Its presence is a Wi-Fi
recovery path, not a reason to alter the proven soldered wiring.
