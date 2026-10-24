# Asus Molly Player: Home key starts a fixed package name

Tested on an Asus "Molly Player" (`ro.product.device=molly`), Android 6.0.1 / API 23, build
`molly-user 6.0.1 MHC19J 20210203`. This is a modified Amlogic/FiberHome set-top box: it ships
rooted (SuperSU), with several preloaded streaming apps and a reseller launcher.

## What the firmware does

- `getprop ro.build.version.sdk` is 23, below upstream's `minSdk` 26. The upstream APK answers
  `INSTALL_FAILED_OLDER_SDK`; this fork's build is the one that installs.
- Pressing Home logs
  `START u0 {act=android.intent.action.MAIN ... cmp=com.fiberhome.fhlauncher_cuxj/.LauncherActivity}`
  from `uid 1000`, but `pm path com.fiberhome.fhlauncher_cuxj` is empty. The package the firmware
  expects is not installed, so the key falls through to whatever was on screen.
- `pm list packages` shows `com.netxeon.lignthome` (the system launcher, `/system/priv-app/launcher.apk`,
  targetSdk 19) and `ca.dstudio.atvlauncher.pro` (ATV Launcher Pro, user-installed). Before the default
  was set, the ordinary `android.intent.action.MAIN` + `android.intent.category.HOME` intent started
  LightHome, which then started ATV Launcher.
- `cmd` does not exist (it arrived with Android 7), so there is no
  `cmd package set-home-activity`, `cmd package query-activities` or `cmd package compile`. The TV's
  Settings app does have a Home picker: `am start -a android.settings.HOME_SETTINGS` opens
  `com.android.settings/.Settings$HomeSettingsActivity` and lists LightHome, ATV Launcher and Beranda
  (this launcher). Choosing Beranda sets the ordinary HOME intent to it.

## Build for API 23

The fork sets `minSdk 23` and enables core-library desugaring for `java.time`. API 24+ calls have
fallbacks for locales, TV input names, network callbacks, changed packages, hardware bitmaps and font
resources. Nothing else about the launcher changes.

Build under the name the firmware starts, install it, and check that the explicit component works:

```bash
python3 -m venv tools/venv
tools/venv/bin/pip install -r tools/requirements.txt
export TV_HOST=<tv ip>

(cd launcher && ./gradlew assembleRelease -PlauncherApplicationId=com.fiberhome.fhlauncher_cuxj)
tools/venv/bin/python tools/tv.py --push launcher/app/build/outputs/apk/release/app-release.apk /data/local/tmp/tvhome.apk
tools/venv/bin/python tools/tv.py 'pm install -r /data/local/tmp/tvhome.apk'
tools/venv/bin/python tools/tv.py 'am start -W -n com.fiberhome.fhlauncher_cuxj/com.gathin.tvhome.MainActivity'
```

The Java namespace stays `com.gathin.tvhome`. `com.fiberhome.fhlauncher_cuxj/.MainActivity` is wrong;
use `com.gathin.tvhome.MainActivity` after the package name.

The fork's manifest carries an alias for the component the firmware starts:

```xml
<activity-alias
    android:name="com.fiberhome.fhlauncher_cuxj.LauncherActivity"
    android:targetActivity="com.gathin.tvhome.MainActivity"
    android:exported="true" />
```

so the complete handover is:

```bash
tools/venv/bin/python tools/tv.py 'am start -W -n com.fiberhome.fhlauncher_cuxj/com.fiberhome.fhlauncher_cuxj.LauncherActivity'
tools/venv/bin/python tools/tv.py 'input keyevent KEYCODE_HOME'
tools/venv/bin/python tools/tv.py 'dumpsys activity activities' | grep mResumedActivity
```

The last line should name `com.fiberhome.fhlauncher_cuxj/.LauncherActivity`.

Then make it the ordinary Home as well, so the box also opens it after a boot:

```bash
tools/venv/bin/python tools/tv.py 'am start -a android.settings.HOME_SETTINGS'
```

With the D-pad, move to the third row, "Beranda", and press Enter. Check the result:

```bash
tools/venv/bin/python tools/tv.py 'dumpsys package com.fiberhome.fhlauncher_cuxj' | grep -A8 'Preferred Activities'
tools/venv/bin/python tools/tv.py 'am start -a android.intent.action.MAIN -c android.intent.category.HOME'
```

The preferred entry should say `com.fiberhome.fhlauncher_cuxj/com.gathin.tvhome.MainActivity` with
`mAlways=true`, and the HOME intent should bring the launcher to the front.

`launcher/build-and-install.sh` does the same on API 23 and leaves both other launchers enabled:

```bash
TV_HOST=<tv ip> \
LAUNCHER_APPLICATION_ID=com.fiberhome.fhlauncher_cuxj \
HOME_COMPONENT=com.fiberhome.fhlauncher_cuxj/com.fiberhome.fhlauncher_cuxj.LauncherActivity \
launcher/build-and-install.sh
```

## Customising the dock

This fork turns on `tile_menu` in `launcher/app/src/main/res/values/config.xml`, so the dock is edited
from the remote:

- On a grid app, hold OK (or press the remote's Menu key) and choose **Tambah ke dock** ("Add to
  dock"), then the dock place to change and OK. The app moves there; the app it replaced moves to the
  grid place it came from.
- On a dock app, the same menu offers **Ganti dengan aplikasi lain** ("Replace with another app"):
  pick the dock place, then any app on the grid. This is how an app that is not in the dock gets in.
- **Pindahkan** ("Move") starts reorder mode; Back saves and leaves it.
- **Setel Dock** in the Settings button's long-press menu starts the same edit mode, place first.

The dock has as many places as the grid has columns: 4 by default, changeable to 5, 6 or 7 in Launcher
settings. The choice is saved in `tvhome-order` and survives a reboot and an app update; a rebuild or a
theme push that changes `dock_apps` resets it to the new list.

## Verified

- The launcher draws its clock, greeting, dock and grid on the TV; screenshots were pulled with
  `tools/tv.py --screenshot`.
- D-pad Down moves from the dock to the grid, Right moves tile to tile; Home returns to the launcher.
- The installed package is `com.fiberhome.fhlauncher_cuxj`, version 2.7. The stock LightHome and ATV
  Launcher stay enabled and were not disabled.
- `Settings$HomeSettingsActivity` lists Beranda, and choosing it writes `mAlways=true` for
  `com.fiberhome.fhlauncher_cuxj/com.gathin.tvhome.MainActivity`. The ordinary HOME intent then resumes
  `com.fiberhome.fhlauncher_cuxj/.LauncherActivity` instead of LightHome.
- The Menu key on a focused tile opens the glass tile menu: on a grid app it reads "Tambah ke dock",
  "Pindahkan" and "Info aplikasi"; on a dock app, "Ganti dengan aplikasi lain" instead of the first.
  The menu was opened over ADB and photographed; the dock itself was left unchanged for the owner.

## Limits and recovery

- A cold boot was not repeated because network ADB on this box has to be switched on again by hand
  after a power cycle. The standard HOME intent and the physical Home key were both checked by ADB.
- `pm uninstall com.fiberhome.fhlauncher_cuxj` removes the launcher; the Home key goes back to its
  original, broken state. Nothing on the TV was disabled, so there is no restore list to apply.
- The box is rooted and has SuperSU. None of the steps above used root, and they should not.
