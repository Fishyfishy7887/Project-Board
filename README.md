# Downstairs touchscreen light panel

This repository keeps `bat-light-wall-switch.yaml` unchanged as the known-good
hardware recovery firmware. The deployable `downstairs-light-panel.yaml` retains
that tested ESP32-S3, SPI, I²C, ST7796, FT63X6, PSRAM, inversion, rotation, and
touch-transform configuration while adding the multi-page controller.

Before deployment, replace the three clearly marked entity substitutions at the
top of `downstairs-light-panel.yaml` with the exact `light.*` entity IDs for the
side entrance and two exterior lights. In Home Assistant's ESPHome integration,
enable **Allow the device to perform Home Assistant actions**.

## One-pass Caper Brains deployment

Run this single block on **Caper Brains** after the pull request has been merged.
It downloads the merged firmware, materializes the locally installed font without
committing it, validates required secret keys without displaying their values,
then performs one config/compile/upload sequence.

```bash
set -e
cd /home/keith/wall-switch-test
curl -fL "https://raw.githubusercontent.com/Fishyfishy7887/Project-Board/main/downstairs-light-panel.yaml" \
  -o downstairs-light-panel.yaml

mkdir -p fonts
FONT_FILE="$(fc-match -f '%{file}\n' 'DejaVu Sans:style=Bold' | head -n 1)"
test -f "$FONT_FILE"
cp -f "$FONT_FILE" fonts/DejaVuSans-Bold.ttf

python3 - <<'PY'
from pathlib import Path
import re
text = Path("secrets.yaml").read_text()
for key in ("wifi_ssid", "wifi_password", "fallback_ap_password"):
    match = re.search(rf"(?m)^\s*{key}:\s*(?:['\"]([^'\"]+)['\"]|([^#\s][^#]*?))\s*(?:#.*)?$", text)
    value = next((group.strip() for group in match.groups() if group), "") if match else ""
    if not value:
        raise SystemExit(f"secrets.yaml: missing or empty {key}")
print("secrets.yaml: required values are present (values hidden)")
PY

.venv/bin/esphome config downstairs-light-panel.yaml
.venv/bin/esphome compile downstairs-light-panel.yaml
.venv/bin/esphome upload downstairs-light-panel.yaml --device /dev/ttyUSB0
```

The ignored `fonts/*.ttf` file is intentionally created only on Caper Brains.
The fallback access point is **Downstairs Panel Setup** and uses
`fallback_ap_password` from `secrets.yaml`.
