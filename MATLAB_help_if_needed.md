# MATLAB on Arch Linux (R2025a) — Working Setup Guide

This is a practical install + fix guide for getting MATLAB **R2025a** running on Arch Linux when it crashes on startup with errors like:

- `Segmentation violation`
- `std::exception::what: Transport stopped`
- stack traces mentioning:
  - `libmwinstall_activationwsclientimpl.so`
  - `libmwlmgrimpl.so`
  - `lc_new_job`

---

## 1) Install MATLAB normally first

Install from the official MathWorks installer as usual.

Example source layout used here:

- Installer media: `/run/media/<user>/MATHWORKS_R2025A`
- Final install path: `/home/<user>/Downloads/R2025a`

---

## 2) Main compatibility fix (important): pin GNU TLS 3.8.8 inside MATLAB

Recent Arch `gnutls` versions can crash MATLAB licensing startup on some systems.

Run:

```bash
mkdir -p /tmp/gnutls-3.8.8
curl -L --fail -o /tmp/gnutls-3.8.8.pkg.tar.zst \
  https://archive.archlinux.org/packages/g/gnutls/gnutls-3.8.8-1-x86_64.pkg.tar.zst
tar -xf /tmp/gnutls-3.8.8.pkg.tar.zst -C /tmp/gnutls-3.8.8 usr/lib

MATROOT="/home/<user>/Downloads/R2025a"
mkdir -p "$MATROOT/bin/glnxa64/gnutls"
cp -af /tmp/gnutls-3.8.8/usr/lib/libgnutls.so.30* "$MATROOT/bin/glnxa64/gnutls/"

cd "$MATROOT/bin/glnxa64"
ln -sfn gnutls/libgnutls.so.30 libgnutls.so.30
ln -sfn gnutls/libgnutls.so.30.40.2 libgnutls.so.30.40.2
```

> If that final version suffix differs on another machine, adjust the symlink target to whatever file exists in `bin/glnxa64/gnutls/`.

---

## 3) ServiceHost executable-stack fix (Arch/glibc issue)

On Arch, MATLAB ServiceHost libs may require stack flag patching.

```bash
BASE="$HOME/.MathWorks/ServiceHost/-mw_shared_installs"
F1="$(find "$BASE" -type f -name libmwfoundation_crash_handling.so | head -n1)"
F2="$(find "$BASE" -type f -name libmwmshrcfservice.so | head -n1)"

# If files are read-only, temporarily add write bit:
chmod u+w "$F1" "$F2"
patchelf --clear-execstack "$F1"
patchelf --clear-execstack "$F2"
chmod u-w "$F1" "$F2"
```

Optional check:

```bash
readelf -W -l "$F1" | grep GNU_STACK
readelf -W -l "$F2" | grep GNU_STACK
```

Expected: `RW` (not `RWE`).

---

## 4) Launch command that worked

Use this launcher command:

```bash
env LD_PRELOAD=/usr/lib/libstdc++.so.6:/usr/lib/libfontconfig.so.1:/usr/lib/libfreetype.so.6 \
  /home/<user>/Downloads/R2025a/bin/matlab -desktop
```

---

## 5) Desktop file (recommended)

Path:

`~/.local/share/applications/matlab.desktop`

Minimal working `Exec` line:

```ini
Exec=env LD_PRELOAD=/usr/lib/libstdc++.so.6:/usr/lib/libfontconfig.so.1:/usr/lib/libfreetype.so.6 /home/<user>/Downloads/R2025a/bin/matlab -desktop
```

---

## 6) Fast troubleshooting checklist

If MATLAB still crashes:

1. Confirm MATLAB is loading local `libgnutls.so.30`:

```bash
MATROOT="/home/<user>/Downloads/R2025a"
timeout 10 env LD_DEBUG=libs \
  LD_PRELOAD=/usr/lib/libstdc++.so.6:/usr/lib/libfontconfig.so.1:/usr/lib/libfreetype.so.6 \
  "$MATROOT/bin/matlab" -batch "disp('ok');exit" 2>&1 | grep -E 'libgnutls\.so\.30|calling init'
```

2. If output shows system `/usr/lib/libgnutls.so.30` instead of MATLAB local one, re-check symlinks in:
   - `$MATROOT/bin/glnxa64/libgnutls.so.30`
   - `$MATROOT/bin/glnxa64/libgnutls.so.30.*`
   - `$MATROOT/bin/glnxa64/gnutls/`

3. Re-run ServiceHost patch section (Step 3).

4. Check latest crash file:
   - `~/matlab_crash_dump.*`

---

## 7) Notes from this machine

- Arch Linux, glibc `2.43`
- MATLAB `R2025a` (`25.1.0.2943329`)
- `gnutls 3.8.9` path still crashed for MATLAB startup licensing thread in this setup
- pinning to `gnutls 3.8.8` inside MATLAB tree + ServiceHost stack patch resolved the issue

---

## 8) One-command launch alias (optional)

Add to `~/.bashrc` or `~/.zshrc`:

```bash
alias matlab='env LD_PRELOAD=/usr/lib/libstdc++.so.6:/usr/lib/libfontconfig.so.1:/usr/lib/libfreetype.so.6 /home/<user>/Downloads/R2025a/bin/matlab -desktop'
```

---

## 9) Simulink/SVN error: `libcrypt.so.1` missing

If you see errors like:

- `Error loading ...libmwcmlink_view_svn_extension.so`
- `libcrypt.so.1: cannot open shared object file`

then add `libcrypt.so.1` compatibility library into MATLAB's `bin/glnxa64`.

```bash
MAT="/home/<user>/Downloads/R2025a/bin/glnxa64"
TMP="/tmp/libxcrypt-compat"

rm -rf "$TMP" && mkdir -p "$TMP"
URL="$(pacman -Sp libxcrypt-compat | tail -n1)"
curl -L --fail -o "$TMP/libxcrypt-compat.pkg.tar.zst" "$URL"
tar -xf "$TMP/libxcrypt-compat.pkg.tar.zst" -C "$TMP" usr/lib

cp -af "$TMP"/usr/lib/libcrypt.so.1* "$MAT"/
[ -e "$MAT/libcrypt.so.1" ] || ln -s libcrypt.so.1.1.0 "$MAT/libcrypt.so.1"
rm -rf "$TMP"
```

Quick validation:

```bash
python - <<'PY'
import ctypes, os
p='/home/<user>/Downloads/R2025a/bin/glnxa64/CmlinkViewExtensions/shared_cmlink/view/svn/libmwcmlink_view_svn_extension.so'
ctypes.CDLL(p, mode=os.RTLD_NOW)
print("LOAD_OK")
PY
```
