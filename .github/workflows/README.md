# KernelSU Build Workflows Guide

> **⚠️ Important Notice:** This is a Custom-build branch. These workflows are provided for testing and evaluation purposes. **Use with caution and at your own risk.** While the workflows have been tested and certified as of the current date, always verify compatibility with your setup before using in production. You assume all responsibility for any outcomes.

This guide explains how to use the automated build workflows in this branch for seamless kernel and manager app compilation.

## Overview

This branch contains optimized GitHub Actions workflows that automate:
- **Kernel compilation** for Android devices
- **Manager APK building** with Kotlin/Jetpack Compose
- **Artifact repacking** with pre-built ksud binaries
- **Telegram notifications** for build status and progress

All workflows are designed to work independently or together, giving you flexibility for different build scenarios.

---

## Workflows Available

### 1. **build-manager.yml** — Manager App & ksud Binaries
**Path:** `.github/workflows/build-manager.yml`

Builds the KernelSU Manager APK and compiles ksud binaries for both ARM64 and ARMv7.

**Triggers:**
- Push to branches: `main`, `dev`, `ci`, `test`, `staging`
- Changes in: `manager/`, `userspace/`, `Cargo.lock`, `Cargo.toml`
- Manual dispatch: `workflow_dispatch`
- Called by other workflows

**Outputs:**
- `manager-gradle` — Signed/unsigned APK from Gradle
- `ksud-aarch64-linux-android` — ARM64 ksud binary
- `ksud-armv7-linux-androideabi` — ARMv7 ksud binary
- `mappings` — ProGuard mapping files (for release builds)

---

### 2. **automated.yml** — Automated Kernel & Manager CI
**Path:** `.github/workflows/automated.yml`

Fully automated build for a specific device with preconfigured defaults (Violet device). Clones kernel source, applies KernelSU patches, compiles, and packages everything into an AnyKernel3 ZIP.

**When to use:**
- Regular scheduled builds
- Continuous integration after upstream syncs
- Predictable, repeatable builds with fixed configuration

**Configuration (in workflow env):**
```yaml
CLANG_VERSION: "clang-r563880c"
KERNEL_SOURCE: "https://github.com/Justadeayo/android_kernel_xiaomi_violet"
KERNEL_BRANCH: "sixteen"
DEFCONFIG_NAME: "vendor/violet-perf_defconfig"
DEVICE_CODENAME: "violet"
KSU_TYPE: "KernelSU"  # or "ReSukiSU-with-susfs"
```

**Outputs:**
- `kernel-zip` — Flashable AnyKernel3 package
- `manager` — Compiled Manager APK
- GitHub Release with both artifacts

**Notifications:**
- Telegram messages for start, success, and failure
- Build duration, file sizes, commit metadata

---

### 3. **Kernel & Manager Build.yml** — Custom Interactive Builds
**Path:** `.github/workflows/Kernel_&_Manger_build.yml`

Manual workflow dispatch for building custom kernel + manager combinations. All parameters are user-supplied. Dynamically targets the calling branch for maximum flexibility.

**When to use:**
- Testing different kernel sources and branches
- Experimenting with different Clang versions
- Building for multiple devices in one session
- Custom defconfigs and build configurations

**Input Parameters:**

| Parameter | Type | Example |
|-----------|------|---------|
| `clang_version` | Choice | `clang-r563880c`, `clang-stable`, `clang-r574158`, `clang-r450784e` |
| `kernel_source` | String | `https://github.com/user/android_kernel_device` |
| `kernel_branch` | String | `main`, `sixteen`, `custom-branch` |
| `defconfig_name` | String | `vendor/violet-perf_defconfig` |
| `device_codename` | String | `violet`, `alioth`, `vayu`, etc. |
| `ksu_type` | Choice | `None`, `KernelSU`, `ReSukiSU-with-susfs` |
| `BUILD_COMMIT` | String | Build notes/description (shown in Telegram) |
| `telegram_chat_id` | String | `-1001234567890` |
| `telegram_bot_token` | String | `123456:ABC-DEF...` |

**How to trigger:**
1. Go to **Actions** → **Kernel & Manager Build**
2. Click **Run workflow**
3. Fill in all parameters
4. Click **Run workflow**

**Outputs:**
- GitHub Release with custom naming
- Telegram notifications with build parameters and results

---

### 4. **release.yml** — Formal Release Publishing
**Path:** `.github/workflows/release.yml`

Publishes GitHub Releases with Manager APK and optional Kernel ZIPs when you push a version tag.

**When to use:**
- Official KernelSU releases
- Publishing tagged versions

**How to trigger:**
```bash
git tag v1.0.0
git push origin v1.0.0
```

**Outputs:**
- GitHub Release with auto-generated release notes
- Attached artifacts: APK + any available kernel ZIP

---

### 5. **sync.yml** — Upstream Synchronization
**Path:** `.github/workflows/sync.yml`

Periodically syncs your `master` branch with the upstream KernelSU repository, then triggers the automated kernel build if changes are detected. Uses dynamic branch reference for flexibility.

**Schedule:**
- Runs daily at: **00:00, 09:00, 17:00 UTC**
- Manual trigger available

**Behavior:**
1. Fetches latest upstream changes
2. If `master` is behind, hard-resets to upstream
3. Pushes updates to `origin/master`
4. Sends Telegram notification
5. Automatically triggers `automated.yml` on the calling branch

---

**And Others...**
Additional workflows and jobs beyond these five provide extended functionality for build management, testing, and distribution.

---

## Quick Start Scenarios

### Scenario A: I want a quick kernel build for Violet
**Use:** `automated.yml`
- Workflow already configured for Violet device
- Just trigger it: **Actions → Automated Build for Violet → Run workflow**
- Check Telegram for progress and results

### Scenario B: I want to test a different kernel source
**Use:** `Kernel & Manager Build.yml`
1. Go to **Actions → Kernel & Manager Build**
2. Click **Run workflow**
3. Fill in your custom kernel repo URL, branch, defconfig
4. Click **Run workflow**
5. Receive Telegram notification when done

### Scenario C: I only want to rebuild the Manager APK
**Use:** `build-manager.yml`
- Automatically triggers on changes to `manager/` or `userspace/`
- Or manually dispatch if you need a rebuild
- Find APK in artifacts: `manager`

### Scenario D: I want to publish an official release
**Use:** `release.yml`
```bash
git tag v1.1.0
git push origin v1.1.0
```
- GitHub Release auto-creates with APK and kernel ZIP (if available)

### Scenario E: I want to sync with upstream daily and build
**Use:** `sync.yml`
- Already scheduled 3 times daily
- Or manually trigger to sync immediately
- Automatically runs `automated.yml` after sync

---

## Build Artifacts & Access

### Where to find built files:

**GitHub UI:**
1. Go to **Actions** tab
2. Select the completed workflow run
3. Scroll to **Artifacts** section
4. Download `manager`, `kernel-zip`, etc.

**Artifact Names:**
- `manager` — Final Manager APK (ready to install)
- `manager-gradle` — Unsigned Gradle APK (intermediate)
- `kernel-zip` — Flashable kernel package
- `ksud-aarch64-linux-android` — ARM64 ksud binary
- `ksud-armv7-linux-androideabi` — ARMv7 ksud binary
- `mappings` — ProGuard obfuscation mappings (release only)

---

## Configuration & Secrets

### Required Secrets (for Telegram notifications)

Add these to **Settings → Secrets and variables → Actions:**

| Secret | Value | Example |
|--------|-------|---------|
| `TELEGRAM_BOT_TOKEN` | Telegram bot token | `123456:ABCDEFghijklmnop...` |
| `TELEGRAM_CHAT_ID` | Target chat ID | `-1001234567890` (for groups, include `-100`) |

**How to get them:**
1. **Bot Token:** Talk to [@BotFather](https://t.me/botfather) on Telegram
2. **Chat ID:** Use [@userinfobot](https://t.me/userinfobot) or message your bot and check logs

### Keystore (for signing APK)

For **non-PR builds**, the Manager uses dummy keystore credentials (see `build-manager.yml`). 

For **production releases**, you may want to supply your own keystore:
1. Create a keystore file
2. Encode it: `base64 -w 0 my.keystore > encoded.txt`
3. Add as GitHub secret: `KEYSTORE_FILE`
4. Update `build-manager.yml` to use it

---

## Understanding the Build Process

### Manager Build Flow
```
build-manager.yml
  ├─ generate-key (creates PR signing key if needed)
  ├─ build-lkm (compiles kernel module)
  ├─ build-ksuinit (compiles ksuinit binary)
  ├─ build-ksud (compiles ksud for ARM64 + ARMv7)
  ├─ build-manager (Gradle assembleRelease)
  └─ repack-manager (injects binaries, re-signs APK)
```

### Kernel Build Flow (automated.yml / custom-build)
```
1. Clone kernel source
2. Download AOSP Clang toolchain
3. Apply KernelSU patches (setup.sh)
4. Modify defconfig with KSU flags
5. Compile with make -j$(nproc)
6. Copy Image.gz + DTBs
7. Package with AnyKernel3
8. Create GitHub Release
9. Send Telegram notification
```

---

## Dynamic Branch Reference

These workflows use dynamic branch references (`${{ github.ref_name }}`) for maximum flexibility:

- **Kernel & Manager Build.yml:** Automatically targets the branch that triggered the workflow (no hardcoded branch names)
- **sync.yml:** Dynamically references the current branch when triggering `automated.yml`

This allows you to:
- Run workflows from any branch without modification
- Test on feature branches safely
- Maintain a single workflow configuration for multiple branches

---

## Troubleshooting

### Build fails with "Cannot find kernel source"
- **Cause:** Kernel repo URL or branch is wrong
- **Fix:** Verify URL and branch exist on GitHub; use HTTPS URLs

### Manager APK is unsigned or won't install
- **Cause:** Running on a PR build without proper keystore
- **Fix:** Use on main branches or supply your own keystore via secrets

### No Telegram notifications
- **Cause:** Bot token or chat ID is invalid/missing
- **Fix:** Verify secrets in **Settings → Secrets and variables → Actions**

### Kernel compilation fails ("olddefconfig" issues)
- **Cause:** Defconfig doesn't exist or symbol dependencies broken
- **Fix:** Verify defconfig path matches your kernel source

### Out of disk space during build
- **Cause:** Clang toolchain + kernel source are large
- **Fix:** Workflow includes SWAP setup; monitor via logs

### APK or Kernel ZIP not in artifacts
- **Cause:** Build failed silently or naming mismatch
- **Fix:** Check job logs in **Actions → [Run] → [Job]** for error messages

### Workflow calls wrong branch
- **Cause:** Hardcoded branch reference instead of dynamic
- **Fix:** Use `${{ github.ref_name }}` instead of hardcoded branch names

---

## Advanced: Customizing Workflows

### Change Telegram notification recipients
Edit the workflow and update hardcoded chat IDs, or pass as secret:

```yaml
TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
```

### Add a new device preset
1. Duplicate `automated.yml`
2. Rename to e.g. `automated-alioth.yml`
3. Change `DEVICE_CODENAME`, `DEFCONFIG_NAME`, etc.
4. Update `on` trigger or add manual dispatch inputs

### Customize Clang version
In `automated.yml` or manual workflow, change:
```yaml
CLANG_VERSION: "clang-r563880c"  # Change this
```

Available versions: `clang-r563880c`, `clang-r574158`, `clang-r450784e`, `clang-stable`

### Skip specific build steps
Add conditions to any step:
```yaml
- name: Some Step
  if: ${{ github.event_name == 'workflow_dispatch' }}
  run: ...
```

### Use dynamic branch references
When calling workflows from other workflows, use:
```yaml
uses: Justadeayo/KernelSU/.github/workflows/build-manager.yml@${{ github.ref_name }}
```

Instead of hardcoding:
```yaml
uses: Justadeayo/KernelSU/.github/workflows/build-manager.yml@test
```

---

## Workflow Status & Monitoring

### View workflow status:
1. Go to **Actions** tab
2. Select workflow (e.g., "Automated Build for Violet")
3. Click the latest run to see real-time logs

### Interpret badge colors:
- 🟢 **Green** — All steps passed
- 🟡 **Yellow** — Running
- 🔴 **Red** — Failed (check logs)

### Check build logs for errors:
1. Click the failed run
2. Click the failed job (e.g., "Kernel_CI")
3. Expand the failed step and read error messages

---

## Performance Tips

### Reduce build time:
- Use `clang-stable` (smallest download)
- Reduce SWAP if you have enough RAM: `swap-size-gb: 8`
- Cache toolchains between runs (currently manual; consider setup-cache action)

### Optimize artifact storage:
- Workflows auto-expire artifacts after 30 days
- Manually delete old runs to save space
- Only upload essential artifacts (disable `mappings` if not needed)

---

## Branch-Specific Notes

This README applies to the **Custom-build** branch (and compatible branches like **test**).

**⚠️ Use with Caution:** While these workflows have been tested and certified as of the current date, they are provided on an **as-is basis**. You assume all responsibility for:
- Testing thoroughly before production use
- Compatibility with your specific hardware and setup
- Any bricked devices or failed builds
- Data loss or other adverse outcomes

The workflows here are provided for testing and evaluation purposes. Always verify compatibility with your setup before using in production.

**Branch comparison:**
- **master:** Upstream-synced, uses official KernelSU setup (stable)
- **test:** Custom build workflows (testing phase, use with caution)
- **custom-build:** Extended custom workflows (tested and certified, use at your own risk)
- **dev:** Development features (not to be used as its unstable and intentionally outdated so as to test some experimentation)

---

## Support & Contributing

**Issues or improvements?**
- Report via GitHub Issues with workflow name and error logs
- Include device info and kernel source details
- Note: Support is provided on best-effort basis

**Want to add a new workflow?**
- Follow the naming convention: `workflow-purpose.yml`
- Document triggers, inputs, and outputs
- Test on a non-main branch first
- Ensure dynamic references where applicable

---

## Disclaimer

By using these workflows, you acknowledge:
- ✅ You understand the experimental nature of this branch
- ✅ You will test thoroughly before production use
- ✅ You assume all risks and responsibility for outcomes
- ✅ The authors and maintainers are not liable for any damages
- ✅ You have read and accepted the `LICENSE`

For more information, see `LICENSE` in the repository root.

---

## Version History

| Version | Date | Notes | Branch |
|---------|------|-------|--------|
| 1.1 | 2026-07-26 | Added custom-build branch support, dynamic references, risk disclaimers | custom-build, test |
| 1.0 | 2026-07-20 | Initial guide for automated build workflows | test |

---

**Last Updated:** 2026-07-26  
**Primary Branches:** Custom-build  
**Status:** ⚠️ Testing Phase — Use with Caution and at Your Own Risk
