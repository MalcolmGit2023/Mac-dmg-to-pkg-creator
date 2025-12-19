# DMG to PKG Converter for macOS

A simple Bash script to **convert DMG installers into PKG packages** for macOS deployment tools like **Jamf Pro** or **Munki**.

## Features

*   Mounts DMG using `hdiutil`.
*   Detects:
    *   Embedded **.pkg/.mpkg** → copies or re-signs.
    *   **.app bundles** → builds a component PKG with `pkgbuild`.
    *   Payload folders (e.g., `/Applications`, `/Library`) → builds a root PKG.
*   Optional **code signing** with Developer ID Installer certificate.
*   Cleans up temporary mounts automatically.

## Usage

```bash
sudo ./dmg_to_pkg.sh <input_dmg> <identifier> <version> <output_pkg> [--sign "Developer ID Installer: Your Name (TEAMID)"]
```

### Examples

```bash
# Convert DMG with .app inside to PKG
sudo ./dmg_to_pkg.sh "/tmp/MyApp.dmg" "com.example.myapp" "1.0.0" "/tmp/MyApp.pkg"

# Convert DMG and sign PKG
sudo ./dmg_to_pkg.sh "/tmp/MyApp.dmg" "com.example.myapp" "1.0.0" "/tmp/MyApp.pkg" --sign "Developer ID Installer: ACME Inc (ABCDE12345)"
```

## Requirements

*   macOS with **Xcode Command Line Tools** installed.
*   For signing: valid **Developer ID Installer** certificate in your keychain.

## Notes

*   Test PKG before deployment:
    ```bash
    sudo installer -pkg /path/to/pkg -target /
    ```
*   Verify signature:
    ```bash
    pkgutil --check-signature /path/to/pkg
    ```

## License

MIT License – free to use and modify.

***

Would you like me to **also include a section for Jamf Pro integration** (how to upload and deploy the generated PKG) in this README? Or keep it minimal for general use?


Below is a **general, unbranded** Bash script you can use to **transform a DMG into an installable PKG** on macOS. It handles common cases:

*   DMG contains a **.pkg/.mpkg** → wraps/renames or copies directly to output
*   DMG contains an **.app** → builds a component PKG using `pkgbuild`
*   DMG contains a **payload folder** (e.g., `Applications/...` files) → builds a PKG from a root directory
*   Optional: **sign** the PKG if you have a Developer ID Installer certificate

***

## Script: `dmg_to_pkg.sh`

> Save to a working directory and run with `sudo`.  
> Example:
>
> ```bash
> sudo ./dmg_to_pkg.sh "/path/to/AppInstaller.dmg" "com.example.app" "1.0.0" "/path/to/output/AppInstaller.pkg"
> ```

```bash
#!/bin/bash
#
# dmg_to_pkg.sh
# Purpose: Convert the contents of a DMG into an installable PKG for deployment (e.g., Jamf).
# Supports: .pkg/.mpkg passthrough, .app component packaging, or building from a payload root.
# Usage:
#   sudo ./dmg_to_pkg.sh <input_dmg> <identifier> <version> <output_pkg> [--sign "Developer ID Installer: Your Name (TEAMID)"]
#
# Examples:
#   sudo ./dmg_to_pkg.sh "/tmp/MyApp.dmg" "com.example.myapp" "1.2.3" "/tmp/MyApp.pkg"
#   sudo ./dmg_to_pkg.sh "/tmp/MyApp.dmg" "com.example.myapp" "1.2.3" "/tmp/MyApp.pkg" --sign "Developer ID Installer: ACME Inc (ABCDE12345)"
#
set -euo pipefail

# --------------- Arguments ---------------
if [[ $# -lt 4 ]]; then
  echo "Usage: $0 <input_dmg> <identifier> <version> <output_pkg> [--sign \"Developer ID Installer: Name (TEAMID)\"]"
  exit 2
fi

INPUT_DMG="$1"
PKG_ID="$2"        # e.g., com.example.app
PKG_VERSION="$3"   # e.g., 1.0.0
OUTPUT_PKG="$4"

SIGNING=""
if [[ $# -ge 5 && "$5" == "--sign" ]]; then
  SIGNING="$6"
fi

# --------------- Sanity checks ---------------
if [[ ! -f "$INPUT_DMG" ]]; then
  echo "ERROR: DMG not found: $INPUT_DMG"
  exit 1
fi

WORKDIR="$(mktemp -d /tmp/dmg2pkg.XXXXXX)"
MOUNTPOINT="$(mktemp -d /tmp/dmgmount.XXXXXX)"
cleanup() {
  set +e
  # Detach DMG if mounted
  if mount | grep -q "$MOUNTPOINT"; then
    hdiutil detach "$MOUNTPOINT" -quiet || true
  fi
  rm -rf "$WORKDIR" "$MOUNTPOINT"
}
trap cleanup EXIT

echo "==> Attaching DMG: $INPUT_DMG"
# Attach without auto-opening, no verification for speed; consider adding -verify in stricter environments
hdiutil attach "$INPUT_DMG" -mountpoint "$MOUNTPOINT" -nobrowse -quiet

# Discover contents
echo "==> Scanning mounted volume: $MOUNTPOINT"
PKG_INSIDE="$(/usr/bin/find "$MOUNTPOINT" -maxdepth 3 -type f \( -name "*.pkg" -o -name "*.mpkg" \) -print -quit)"
APP_INSIDE="$(/usr/bin/find "$MOUNTPOINT" -maxdepth 3 -type d -name "*.app" -print -quit)"
PAYLOAD_ROOT=""
if [[ -d "$MOUNTPOINT/Applications" || -d "$MOUNTPOINT/Library" || -d "$MOUNTPOINT/usr" || -d "$MOUNTPOINT/opt" ]]; then
  # Common payload folders at the DMG root
  PAYLOAD_ROOT="$MOUNTPOINT"
fi

# --------------- Paths summary ---------------
echo "==> Detected:"
[[ -n "$PKG_INSIDE"    ]] && echo "    • PKG/MPKG: $PKG_INSIDE"
[[ -n "$APP_INSIDE"    ]] && echo "    • APP:      $APP_INSIDE"
[[ -n "$PAYLOAD_ROOT"  ]] && echo "    • Payload root: $PAYLOAD_ROOT"

# --------------- Case 1: DMG contains PKG/MPKG ---------------
if [[ -n "$PKG_INSIDE" ]]; then
  echo "==> DMG contains an installer package. Copying to $OUTPUT_PKG"
  # If the DMG includes a valid pkg, we can either:
  #  - copy it directly, or
  #  - wrap/re-sign with productbuild (optional)
  cp -f "$PKG_INSIDE" "$OUTPUT_PKG"

  # Optional re-sign (for consistency with org policy)
  if [[ -n "$SIGNING" ]]; then
    echo "==> Rebuilding (productbuild) with signing: $SIGNING"
    TMP_PKG="$WORKDIR/rebuilt.pkg"
    productbuild --sign "$SIGNING" --package "$OUTPUT_PKG" "$TMP_PKG"
    mv -f "$TMP_PKG" "$OUTPUT_PKG"
  fi

  echo "==> Done: $OUTPUT_PKG"
  exit 0
fi

# --------------- Case 2: DMG contains an APP bundle ---------------
if [[ -n "$APP_INSIDE" ]]; then
  echo "==> Building component PKG from app: $APP_INSIDE"
  # Component packaging tells pkgbuild to place the .app into /Applications
  # If the app is meant for a different location, use --install-location accordingly.
  # Note: This will not run postinstall scripts; add them via --scripts if needed.
  COMPONENT_PKG="$WORKDIR/component.pkg"
  if [[ -n "$SIGNING" ]]; then
    pkgbuild \
      --component "$APP_INSIDE" \
      --install-location "/Applications" \
      --identifier "$PKG_ID" \
      --version "$PKG_VERSION" \
      --sign "$SIGNING" \
      "$COMPONENT_PKG"
  else
    pkgbuild \
      --component "$APP_INSIDE" \
      --install-location "/Applications" \
      --identifier "$PKG_ID" \
      --version "$PKG_VERSION" \
      "$COMPONENT_PKG"
  fi

  # (Optional) Wrap in distribution with productbuild (useful for future multi-component builds)
  productbuild --package "$COMPONENT_PKG" "$OUTPUT_PKG"
  echo "==> Done: $OUTPUT_PKG"
  exit 0
fi

# --------------- Case 3: Build from payload directories ---------------
if [[ -n "$PAYLOAD_ROOT" ]]; then
  echo "==> Building PKG from payload root: $PAYLOAD_ROOT"
  # Copy payload to a staging root to avoid including DMG metadata
  STAGING_ROOT="$WORKDIR/root"
  mkdir -p "$STAGING_ROOT"
  rsync -a --exclude=".DS_Store" --exclude="*.VolumeIcon.icns" "$PAYLOAD_ROOT/" "$STAGING_ROOT/"

  # If you need scripts, place a Scripts/ directory with preinstall/postinstall into WORKDIR/scripts
  SCRIPTS_DIR="$WORKDIR/scripts"
  if [[ -d "$SCRIPTS_DIR" ]]; then
    SCRIPT_FLAG=(--scripts "$SCRIPTS_DIR")
  else
    SCRIPT_FLAG=()
  fi

  if [[ -n "$SIGNING" ]]; then
    pkgbuild \
      --root "$STAGING_ROOT" \
      --identifier "$PKG_ID" \
      --version "$PKG_VERSION" \
      --install-location "/" \
      "${SCRIPT_FLAG[@]}" \
      --sign "$SIGNING" \
      "$OUTPUT_PKG"
  else
    pkgbuild \
      --root "$STAGING_ROOT" \
      --identifier "$PKG_ID" \
      --version "$PKG_VERSION" \
      --install-location "/" \
      "${SCRIPT_FLAG[@]}" \
      "$OUTPUT_PKG"
  fi

  echo "==> Done: $OUTPUT_PKG"
  exit 0
fi

# --------------- No supported content found ---------------
echo "ERROR: No .pkg/.mpkg, .app, or recognizable payload folders found in DMG."
exit 3
```

***

## How it works

1.  **Attach DMG**: Uses `hdiutil attach` to mount the image quietly.
2.  **Detect content**:
    *   Finds the first **`.pkg` or `.mpkg`** up to 3 levels deep.
    *   If none, looks for **`.app`** up to 3 levels deep.
    *   If none, checks for common **payload roots** (`Applications`, `Library`, `usr`, `opt`) at the mount root.
3.  **Build/Copy**:
    *   **PKG inside DMG** → copies it directly to `OUTPUT_PKG` (optionally re-signs).
    *   **APP inside DMG** → builds a **component PKG** to `/Applications`.
    *   **Payload folders** → builds a PKG with `--root` to install to `/`.
4.  **Optional signing**: If you pass `--sign "Developer ID Installer: … (TEAMID)"`, the resulting PKG will be signed.

***

## Common scenarios

*   **Vendor DMG contains a PKG**  
    Use Case: Microsoft, Adobe installers often ship PKG inside DMG.  
    Result: Script copies the PKG out (optionally re-signs).

*   **App-only DMG (drag-to-Applications)**  
    Use Case: Tools that expect users to drag the `.app` to Applications.  
    Result: Script creates a proper **component PKG** installing the app to `/Applications`.

*   **Payload DMG**  
    Use Case: DMG has pre-laid folder structure for `/Library`, `/usr`, etc.  
    Result: Script builds a PKG from the payload root.

***

## Jamf Pro deployment

*   Package your resulting `*.pkg` and upload to **Jamf Admin** (or Jamf Pro Packages).
*   Create a **Policy**:
    *   **Packages** payload → add the new PKG.
    *   Scope to desired Smart Groups (e.g., “needs Zulu”, “app not installed”).
*   If using scripts (pre/postinstall), place them under a `Scripts/` directory before running `pkgbuild` and include `--scripts`.

***

## Tips & pitfalls

*   **Component PKG vs. Root PKG**:  
    Use `--component` for app bundles destined for `/Applications`. Use `--root` for arbitrary files/directories that need to land elsewhere.
*   **Relocatable apps**: Some apps expect to live in non-standard paths; adjust `--install-location`.
*   **Signing**:
    *   Requires Xcode command line tools and a valid **Developer ID Installer** cert.
    *   Verify with: `pkgutil --check-signature <pkg>`
*   **Testing**:
    *   Dry-run on a test Mac: `sudo installer -pkg /path/to/pkg -target /`
    *   Inspect payload: `pkgutil --expand-full <pkg> /tmp/expanded`

***

## Optional: minimal “component-only” variant

If you only ever need to convert **APP-in-DMG → PKG**:

```bash
sudo ./dmg_to_pkg.sh "/path/to/App.dmg" "com.example.app" "1.0.0" "/path/to/App.pkg"
```

The script will detect the `.app` and produce a signed (optional) PKG to `/Applications`.

