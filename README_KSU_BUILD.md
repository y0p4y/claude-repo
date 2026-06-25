# KernelSU-Next + SuSFS on Pixel 4a 5G (bramble) — build & flash guide

Goal: replace the userspace Magisk root (which leaves the magiskd socket in
`/proc/net/unix` → AppSealing `[20064]/[50025]`) with a **KernelSU-Next + SuSFS**
kernel that hides root at the **kernel level** — no libc hooks (→ no `[50040]`)
and no socket/mount/prop artifacts (→ no `[20064]/[50025]`). Then M-Smile's
RASP check passes, the app runs **fully and indefinitely**, and you can navigate
to dormant/registration while logcat (`ReactNativeJS`) captures `ac`/`sk`+bodies.

## Why this can't be done on the Kali box
- bramble = **non-GKI kernel 4.19** → no prebuilt KSU+SuSFS image exists; the
  kernel must be **compiled**.
- The Kali host is aarch64 with 11 GB free / 2.8 GB RAM and no GitHub CLI — it
  cannot run the x86_64 AOSP kernel toolchain or a 30–50 GB build.
- So: **build on GitHub Actions** (free x86_64 runners), **flash from your PC**.

## Device facts (read from the device)
- `bramble`, build `SQ3A.220705.003.A1`, Android 12, kernel `4.19.224`
- bootloader **unlocked** (`verifiedbootstate=orange`), active slot `_b`
- Recovery backup already taken: `out/dynamic/recovery/boot_b_magisk_backup.img`
  (sha256 `02207e889cb51a796a231a68fe69a02a7e1476d0730fe15ea7f0518c5768b6e7`)

---

## STEP 1 — build the kernel (GitHub Actions)
1. Create a new GitHub repo (e.g. `redbull-ksu`).
2. Copy this folder's `.github/workflows/build.yml` into it and push:
   ```bash
   git init && git add .github/workflows/build.yml && git commit -m init
   git branch -M main && git remote add origin git@github.com:<you>/redbull-ksu.git
   git push -u origin main
   ```
3. GitHub → repo → **Actions** → "build-redbull-ksunext-susfs" → **Run workflow**.
   - `manifest_branch`: start with `android-msm-redbull-4.19-android12` (matches
     this device's Android 12). If the build fails on a missing branch, the valid
     redbull-4.19 branches are `-android12`, `-android12L`, `-android13`,
     `-android13-qpr3`; pick the one matching the Android version you run.
   - `susfs_branch`: `kernel-4.19` (from gitlab.com/simonpunk/susfs4ksu).
4. When green, download artifact **AnyKernel3-redbull-ksunext-susfs** (and
   `kernel-image`). NOTE: the build is unverified end-to-end — a SuSFS patch hunk
   or build-script name may need a 1-line tweak; the workflow logs say which.

## STEP 2 — flash (from your PC; recovery image ready)
Two ways; **A** needs no custom recovery.

**A) Repack stock boot + flash (recommended)**
1. Download the factory image for `SQ3A.220705.003.A1` from
   https://developers.google.com/android/images#bramble , unzip, and from the
   inner `image-bramble-*.zip` extract `boot.img` (stock).
2. With magiskboot (from the Magisk app, or `unzip` the kernel-image), replace the
   kernel in stock boot:
   ```bash
   ./magiskboot unpack boot.img
   cp Image.lz4-dtb kernel        # the artifact (rename as needed)
   ./magiskboot repack boot.img boot-ksu.img
   ```
3. Flash:
   ```bash
   adb reboot bootloader
   fastboot flash boot boot-ksu.img
   fastboot reboot
   ```

**B) AnyKernel3 via temporary recovery**
```bash
adb reboot bootloader
fastboot boot twrp-or-orangefox-bramble.img      # temporary, not flashed
# in recovery: adb sideload AnyKernel3.zip
```

**If anything goes wrong → restore the working device:**
```bash
adb reboot bootloader
fastboot flash boot ../recovery/boot_b_magisk_backup.img   # back to current Magisk root
fastboot reboot
```
(Full brick-proofing: `fastboot --slot=b flashall` the stock factory image.)

## STEP 3 — post-flash: KernelSU + SuSFS config
1. Install the **KernelSU-Next manager** APK (github.com/KernelSU-Next/KernelSU-Next
   releases). Open it → should show "Working".
2. Remove/disable Magisk root for this kernel (KSU replaces it). Keep the Magisk
   *app* uninstalled or hidden (AppSealing enumerates packages).
3. Install the **SuSFS module** (ksu_susfs) + set it to hide for the target:
   - add `com.bankmega.msmiledev` (+ `:ip_service`) to KSU's denylist/umount list,
   - `susfs add_sus_path /data/adb`, hide `/debug_ramdisk`, spoof uname, etc.
     (the susfs module's `service.sh` usually does the standard set; verify with
     `ksu_susfs show_enabled_features`).
4. Hide the KSU manager app (rename package) — AppSealing flags known root managers.

## STEP 4 — verify the hide worked, then capture
```bash
# the magiskd socket MUST be gone now (this was the [20064]/[50025] trigger):
adb shell "su -c 'cat /proc/net/unix'" | grep -i magisk      # expect: nothing
# launch the app and confirm it stays up + RN runs:
adb logcat -c
adb shell monkey -p com.bankmega.msmiledev -c android.intent.category.LAUNCHER 1
adb logcat -s ReactNativeJS:V | tee PIXEL_ksu_capture.log
```
If AppSealing no longer kills (no `Kill Process [...]`, no ANR) the app stays
alive — navigate to the dormant-activation / registration flow and the `curl`
lines with `ac`/`sk` + bodies + responses are captured in `PIXEL_ksu_capture.log`.

## Honesty note
SuSFS is purpose-built to defeat exactly this detection class (mounts, props,
`/proc` entries, root manager). High likelihood it makes AppSealing's check pass.
Not 100% — CoVault could have an additional signal; if a specific check still
fires, the kernel-level approach still gives far more to work with (and SuSFS has
knobs for most). The `ROOT_BYPASS_POC.md` documents what each kill code means so
you can map any residual detection.
