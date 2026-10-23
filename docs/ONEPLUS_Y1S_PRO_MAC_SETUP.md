# Turn an Android TV into a clean Apple TV-style home — full Mac setup guide

**A beginner-friendly, real-device walkthrough for CleanYourTV**  
**Tested setup:** OnePlus TV 50 Y1S Pro (50-inch), Android TV 10, using macOS and Wi-Fi ADB.  
**Repository to clone:** https://github.com/anuanurag2410/cleanyourtv  
**Original project and credit:** [Gathin23/cleanyourtv](https://github.com/Gathin23/cleanyourtv)

> **Read before starting.** This is a tested *example*, not a universal debloat script. Different Android/Google TVs have different firmware, packages, launchers and remote controls. You are responsible for testing and recovering your own TV. The safest path is: **connect → inspect → back up → build → install → test → select Home → reboot-test → secure ADB**. Root is not required. We do **not** uninstall system packages.

## Contents

1. [What you need](#1-what-you-need)
2. [Find your TV model, Android version and IP address](#2-find-your-tv-model-android-version-and-ip-address)
3. [Enable developer options and debugging](#3-enable-developer-options-and-debugging)
4. [Install ADB on the Mac and connect](#4-install-adb-on-the-mac-and-connect)
5. [Back up first](#5-back-up-first)
6. [Discover the actual device and Home packages](#6-discover-the-actual-device-and-home-packages)
7. [Clone this fork and install build prerequisites](#7-clone-this-fork-and-install-build-prerequisites)
8. [Customize the four apps and Live TV](#8-customize-the-four-apps-and-live-tv)
9. [Build the APK without changing your TV](#9-build-the-apk-without-changing-your-tv)
10. [Install and test the new launcher safely](#10-install-and-test-the-new-launcher-safely)
11. [Make CleanYourTV the default Home](#11-make-cleanyourtv-the-default-home)
12. [Verify after restarting](#12-verify-after-restarting)
13. [Fix HDMI ARC or missing sound](#13-fix-hdmi-arc-or-missing-sound)
14. [Restore the original launcher](#14-restore-the-original-launcher)
15. [Common errors and fixes](#15-common-errors-and-fixes)
16. [What NOT to do and final security checks](#16-what-not-to-do-and-final-security-checks)

---

## 1. What you need

- An **Android TV or Google TV**, its physical remote and permission to change its settings.
- A **Mac** with Terminal and an internet connection for downloading build tools.
- **TV and Mac on the same trusted home network.** Wi-Fi or wired Ethernet on the same LAN is fine.
- Enough TV storage for a small APK, and preferably a stable power supply.
- A cable is **not** needed if your TV exposes and authorizes ADB on the local network. Some models require initial USB pairing or Android's wireless pairing feature.
- Homebrew installed on the Mac: https://brew.sh. To confirm: run <code>brew --version</code>.
- Allow time to test **remote navigation, YouTube/streaming, HDMI inputs, sound and a full reboot** before turning off debugging.

**Reference device in this guide:** OnePlus 50 Y1S Pro, Android TV 10, model identifier reported over ADB as <code>C00</code>, with stock Home package <code>com.google.android.tvlauncher</code>. The example local IP <code>192.168.1.10</code> belonged to that particular TV; **find your own IP, do not blindly copy it**.

Other models (including the repository's OnePlus <code>D00</code> example on Android 14) are **not** the same device.

## 2. Find your TV model, Android version and IP address

**On the TV:**

1. Open **Settings → Device Preferences → About** (menu wording varies).
2. Note the **Model**, **Android TV OS version**, and **Build**. A retail name such as *OnePlus 50 Y1S Pro* may differ from the technical model ID returned by ADB.
3. Open **Settings → Network & Internet → Connected network**.
4. Note the **IP address** (example: <code>192.168.1.10</code>).
5. Keep the TV turned on and connected to the network.

You can also identify the device over ADB in step 6.

## 3. Enable developer options and debugging

1. On the TV open **Settings → Device Preferences → About**.
2. Highlight **Build** and press OK **7 times**, until the developer-options confirmation appears.
3. Return to **Device Preferences → Developer options**.
4. Enable **USB debugging**.
5. If your TV additionally shows **Network debugging / ADB debugging / Wireless debugging**, follow the prompts to enable it.
6. Keep **Don't keep activities** **OFF** and **Background process limit** at **Standard**.

**Note:** Many Android TV 10 models show USB debugging but no separate network debugging switch; some still listen on TCP port 5555. Other TVs do not. Turning on USB debugging does **not** guarantee remote ADB access.

Keep access to the physical remote; the TV may display an **Allow USB debugging?** dialog later. Also note that USB debugging can temporarily affect power supplied to USB peripherals on some TVs.

## 4. Install ADB on the Mac and connect

**Install and verify Android Platform Tools:**

~~~bash
brew install --cask android-platform-tools
adb version
~~~

Choose your **own** TV IP; this variable is used by later commands in this Terminal session:

~~~bash
export TV_IP="192.168.1.10"
~~~

**Check whether the ADB port is reachable (Mac command):**

~~~bash
nc -vz -G 3 "$TV_IP" 5555
~~~

If this says **succeeded**, connect:

~~~bash
adb connect "$TV_IP:5555"
adb devices -l
~~~

When the authorization dialog appears on your TV, choose **Allow**. The device list should show a line ending in **device**, for example:

~~~text
192.168.1.10:5555    device
~~~

- If the device says **unauthorized**, accept the TV's debugging prompt and run <code>adb devices -l</code> again.
- If **connection refused / timeout**, don't repeatedly run disable commands. Your TV might not expose TCP ADB. Check network isolation, IP address and Developer options. On models with a **Wireless debugging** pairing screen, use the displayed pairing IP/port/code with <code>adb pair IP:PAIR_PORT</code>, then connect to its displayed ADB connection port. Some older sets require a physical USB ADB connection first.
- If the connection breaks after reboot, verify the IP address and debugging setting again. On some firmware TCP ADB does not persist across reboot.
- **Only use ADB on your own TV and trusted network.**

## 5. Back up first

The following creates **local configuration snapshots** on your Mac, not a full firmware image. Do **not** upload the backup to GitHub: Android settings can contain sensitive information.

~~~bash
BACKUP_DIR="$HOME/Desktop/OnePlus-TV-Backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

adb shell pm list packages > "$BACKUP_DIR/packages-all.txt"
adb shell pm list packages -d > "$BACKUP_DIR/packages-disabled-before.txt"
adb shell getprop > "$BACKUP_DIR/device-properties.txt"
adb shell settings list system > "$BACKUP_DIR/settings-system.txt"
adb shell settings list secure > "$BACKUP_DIR/settings-secure.txt"
adb shell settings list global > "$BACKUP_DIR/settings-global.txt"

ls -lh "$BACKUP_DIR"
~~~

Keep this folder. In particular, record which packages were **already disabled** before you make changes, so you do not accidentally enable those during recovery.

## 6. Discover the actual device and Home packages

**Do not guess a package name from a different TV's guide.** Run:

~~~bash
adb shell getprop ro.product.model
adb shell getprop ro.build.version.release
adb shell getprop ro.build.version.sdk

adb shell pm list packages | grep -Ei 'launcher|oneplus|heytap|netflix|amazonvideo|youtube|hotstar|livetv|tvcenter'
~~~

Check the **currently resolved Home activity**:

~~~bash
adb shell cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME
~~~

Check **every registered Home candidate**:

~~~bash
adb shell cmd package query-activities --brief -a android.intent.action.MAIN -c android.intent.category.HOME
~~~

On the tested OnePlus Y1S Pro, the original Home resolved to:

~~~text
com.google.android.tvlauncher/.MainActivity
~~~

Your TV might have **another package** or several Home candidates. Keep a copy of these outputs for recovery. Do not disable a package simply because its name contains "launcher".

## 7. Clone this fork and install build prerequisites

**Download the fork used in this guide:**

~~~bash
cd ~/Desktop
git clone https://github.com/anuanurag2410/cleanyourtv.git
cd cleanyourtv
ls launcher
~~~

If you already cloned it, use your existing folder rather than cloning a duplicate.

**Install Java 17 and the Android SDK Command-Line Tools:**

~~~bash
brew install openjdk@17
brew install --cask android-commandlinetools
~~~

Configure your current Terminal session (works with standard Homebrew paths on Apple Silicon and Intel Macs):

~~~bash
export JAVA_HOME="$(brew --prefix openjdk@17)/libexec/openjdk.jdk/Contents/Home"
export PATH="$JAVA_HOME/bin:$PATH"
export ANDROID_HOME="$(brew --prefix)/share/android-commandlinetools"

java -version
sdkmanager --version
~~~

The Java version should start with **17**. If your Mac normally uses Java 8, this step selects Java 17 for the **current Terminal session**; it does not uninstall Java 8.

**Install the Android 34 platform and matching build tools:**

~~~bash
sdkmanager --sdk_root="$ANDROID_HOME" "platforms;android-34" "build-tools;34.0.0"
~~~

Accept licenses when prompted; if needed:

~~~bash
sdkmanager --sdk_root="$ANDROID_HOME" --licenses
~~~

If <code>sdkmanager</code> is not found, verify the cask installation and use its installed executable from the Android command-line tools. If Java 17 is not selected, re-run the environment exports in the same Terminal window.

The present launcher project compiles with SDK 34 and requires **Android API 26+**. The tested Android TV 10 device meets this requirement; older devices may not.

## 8. Customize the four apps and Live TV

The launcher configuration file is:

**<code>launcher/app/src/main/res/values/config.xml</code>**

The fork's current default dock already uses these app IDs, in this order:

~~~xml
<string-array name="dock_apps">
    <item>com.netflix.ninja</item>
    <item>com.amazon.amazonvideo.livingroom</item>
    <item>com.google.android.youtube.tv</item>
    <item>in.startv.hotstar</item>
</string-array>
~~~

This corresponds to **Netflix → Prime Video → YouTube → Hotstar/JioHotstar**. An app must actually be installed on your TV for its dock tile to appear. Verify installed packages with the command in step 6; names can vary by TV or app release.

**Live TV input configuration matters.** The default example in the upstream config uses <code>com.mediatek.wwtv.tvcenter</code>. The tested OnePlus configuration used <code>com.heytap.tv.livetv</code>. **First verify that your TV really has it:**

~~~bash
adb shell pm path com.heytap.tv.livetv
~~~

If the output begins with <code>package:</code>, you can adapt the *local working copy* of config.xml:

~~~bash
cd ~/Desktop/cleanyourtv
sed -i '' 's/com.mediatek.wwtv.tvcenter/com.heytap.tv.livetv/g' launcher/app/src/main/res/values/config.xml

grep -n 'com.heytap.tv.livetv' launcher/app/src/main/res/values/config.xml
~~~

This replaces occurrences including the Live TV shortcut and any matching hidden-app entry. On a **different TV**, locate the correct tuner app first, then edit <code>live_tv_package</code> accordingly; do not copy the OnePlus setting without verification. If the package is absent, **do not run this replacement**.

For further customization, see the original project's [launcher configuration](../launcher/app/src/main/res/values/config.xml) and [themes](../themes/README.md).

## 9. Build the APK without changing your TV

~~~bash
cd ~/Desktop/cleanyourtv/launcher

export JAVA_HOME="$(brew --prefix openjdk@17)/libexec/openjdk.jdk/Contents/Home"
export PATH="$JAVA_HOME/bin:$PATH"
export ANDROID_HOME="$(brew --prefix)/share/android-commandlinetools"

printf 'sdk.dir=%s\n' "$ANDROID_HOME" > local.properties
./gradlew assembleRelease --console=plain
ls -lh app/build/outputs/apk/release/app-release.apk
~~~

Expected outcome: **BUILD SUCCESSFUL**, followed by the APK file's name and size. The first Gradle build may download dependencies.

**Critical distinction:** This step only builds an APK on your Mac. **Do not run <code>launcher/build-and-install.sh</code> at this stage**: the repository's automated script can disable a stock launcher as part of the handover. Our beginner-safe approach is to install and manually test first.

## 10. Install and test the new launcher safely

Keep the original Android TV launcher enabled.

~~~bash
cd ~/Desktop/cleanyourtv/launcher
adb devices -l
adb install -r app/build/outputs/apk/release/app-release.apk
~~~

Successful installation normally prints:

~~~text
Success
~~~

If the TV requests installation confirmation, approve it **on the TV**. Do not bypass Play Protect or security blocks.

Open the new launcher *manually*:

~~~bash
adb shell am start -W -n com.gathin.tvhome/.MainActivity
~~~

The result should show the activity starting successfully, and the Apple TV-style home should appear.

**Test the physical remote before proceeding:**

- Arrow keys move focus between the four apps; OK launches an app.
- Open YouTube, Netflix or Prime Video; play audio.
- Open the launcher Settings and Inputs/HDMI menu.
- Check sound, volume, Bluetooth devices and any HDMI ARC soundbar.
- Press Home and observe which launcher appears. The old Home may still take priority at this stage — that's normal.

**If the new launcher cannot open, STOP here.** Leave the original launcher alone and troubleshoot the install/build first.

## 11. Make CleanYourTV the default Home

Start with the non-destructive attempt:

~~~bash
adb shell cmd package set-home-activity --user 0 com.gathin.tvhome/.MainActivity

adb shell cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME
~~~

If this resolves to <code>com.gathin.tvhome/.MainActivity</code> and the **physical Home button** works, **do not disable anything else**.

### If the old Home still wins (tested OnePlus Y1S Pro)

In our case, Google TV's stock Home had a higher HOME intent priority than CleanYourTV. Android even reported success when selecting the new default, but the old home still opened.

**Only perform the next sequence if ALL of these are true:**

1. ADB is connected and your physical remote works.
2. CleanYourTV was installed and manually started successfully.
3. Step 6 confirmed the current **stock** HOME package is **exactly** <code>com.google.android.tvlauncher</code>.
4. You have your backup and know the recovery commands in step 14.

Disable *that one verified stock Home package* (this is reversible, not an uninstall):

~~~bash
adb shell pm disable-user --user 0 com.google.android.tvlauncher
~~~

Then assign and verify:

~~~bash
adb shell cmd package set-home-activity --user 0 com.gathin.tvhome/.MainActivity

adb shell cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME
~~~

Expected resolved Home:

~~~text
com.gathin.tvhome/.MainActivity
~~~

Now press the **actual Home key** on your remote. If the activity does not resolve correctly or the remote does not work, **restore the stock launcher immediately using step 14**. Do not reboot into an unverified configuration.

**Other TVs:** The stock Home package may be different, and some Google TV firmware also registers a setup wizard as HOME. Do not apply this OnePlus-specific disable command to other models. Read [upstream troubleshooting](../docs/TROUBLESHOOTING.md) and recover safely rather than disabling unknown system packages.

## 12. Verify after restarting

Once Home, playback, inputs and sound are working:

1. Restart the TV normally.
2. Press **Home** — CleanYourTV should appear.
3. Test YouTube audio, Netflix, Prime Video, Hotstar and HDMI devices.
4. Test the TV volume keys and **your soundbar, if connected**.
5. If needed, reconnect ADB and re-check:

~~~bash
adb connect "$TV_IP:5555"
adb shell pm list packages -d | grep -E 'com.google.android.tvlauncher|com.gathin.tvhome'
adb shell pm list packages -e | grep -E 'com.google.android.tvlauncher|com.gathin.tvhome'
adb shell cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME
~~~

If CleanYourTV is installed but the *old* Home comes back after reboot, check which packages are enabled and which activity resolves first. Do not blindly repeat package-disable commands or permanently automate them. The tested setup eventually survived reboot, but behavior varies by firmware.

## 13. Fix HDMI ARC or missing sound

**Real issue during our OnePlus test:** After switching launchers, a Mivi soundbar connected through HDMI ARC stopped producing audio. At one point even the internal TV speakers were silent. Audio returned after checking the TV's sound-output setting (including **Auto**) and power-cycling/testing. This timing **does not prove** that the launcher disabled or broke an audio service.

**For the OnePlus 50 Y1S Pro only**, OnePlus documents **HDMI 1** as the **eARC/ARC** port:
https://www.oneplus.in/tv-y1s-pro

Follow this order:

1. **Check mute and volume.** Play a YouTube video with TV volume around 25–30.
2. **Check TV sound settings:** Settings → Device Preferences → Sound / Audio Output. Try **TV Speakers/Internal Speakers** to test internal sound. If your menu has **Auto**, test it as well. Menu labels differ across firmware.
3. **Check the Mivi source:** Select **ARC / HDMI TV** using the soundbar's own remote; not Bluetooth/AUX.
4. **Check the ports:** TV **HDMI 1 (ARC/eARC)** → soundbar **HDMI ARC / TV ARC**.
5. **Enable HDMI-CEC / HDMI ARC** in the TV's input/audio settings if available. ARC may not negotiate correctly while CEC is off.
6. **If Auto has no audio**, try **PCM** as a temporary digital-output compatibility test.
7. **Power cycle** both TV and soundbar: turn them off, unplug power, disconnect/reseat HDMI, wait about 1–2 minutes, reconnect, turn on the soundbar then the TV, and try ARC again.
8. **Isolate the problem:** Disconnect the Mivi's HDMI cable; switch audio to internal TV speakers and test YouTube. If internal speakers work but ARC does not, focus on ARC/CEC/cable/source. If internal speakers also fail, focus on output settings and the TV itself.
9. Only if necessary, **restore the original launcher** (step 14) to compare behavior. Do **not** reset TV firmware or delete audio-system packages to troubleshoot sound.

After fixing the sound, test again after a reboot. A successful soundbar test is more important than cosmetic launcher changes.

## 14. Restore the original launcher

If Home is broken, the new launcher freezes, or you want the stock experience back, use these commands **while ADB still works**.

For the **tested** TV whose original launcher was verified as <code>com.google.android.tvlauncher</code>:

~~~bash
adb connect "$TV_IP:5555"
adb shell pm enable --user 0 com.google.android.tvlauncher
adb shell cmd package set-home-activity --user 0 com.google.android.tvlauncher/.MainActivity
adb shell cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME
adb shell input keyevent 3
~~~

Verify that the original Home appears with the physical remote. Android may display a launcher chooser, in which case select the stock Android TV launcher.

**Never use that package name on another TV without first confirming it in step 6.** If the TV is offline, use its own Settings/remote to reconnect Wi-Fi and re-enable debugging before retrying ADB.

Once the original home works, optionally uninstall **only the user-installed CleanYourTV app**:

~~~bash
adb uninstall com.gathin.tvhome
~~~

Do this **only after** restoring and testing the stock launcher. Do not uninstall stock system packages.

For any optional debloat you separately performed with the repository's <code>tools/disable.sh</code>, use its matching <code>tools/restore.sh</code> and original disable list/record, per the [project documentation](../README.md). The steps in *this* guide do **not** require a debloat list.

## 15. Common errors and fixes

| What you see | What it usually means | What to do |
| --- | --- | --- |
| <code>adb: command not found</code> | Platform Tools missing | Install <code>brew install --cask android-platform-tools</code>, reopen Terminal if necessary |
| <code>sdkmanager: command not found</code> | Command-line SDK tools missing or unavailable in PATH | Install Android command-line tools and check <code>sdkmanager --version</code> |
| Java version **8** / Gradle JDK error | Build requires Java **17** | Re-run the <code>JAVA_HOME</code> and <code>PATH</code> exports in step 7 |
| TCP **5555 refused / timeout** | Remote ADB is not listening, wrong IP, or network isolation | Recheck TV IP, LAN, debugging; use supported wireless pairing/USB method instead |
| <code>unauthorized</code> | TV has not trusted this Mac | Approve the prompt on the TV; run <code>adb devices -l</code> again |
| <code>INSTALL_FAILED_OLDER_SDK</code> | Android version below app minimum | Stop; check <code>ro.build.version.sdk</code>. Do not force install |
| App install blocked / verification failure | Firmware or device policy blocks sideloading | Respect the block; don't bypass security checks |
| Launcher opens manually but Home opens stock | Stock launcher has higher priority or default wasn't retained | Run Home diagnostics in steps 6 and 11; only disable the *verified* stock package after testing |
| Old launcher reappears after reboot | Home default or enabled packages changed | Check ADB package lists and resolved activity in step 12 |
| No sound / Mivi HDMI ARC silent | Audio output, CEC, handshake, source or TV settings | Follow step 13 before altering packages |
| Blank app dock tile | Package ID absent or changed | Use <code>pm list packages</code> and edit <code>dock_apps</code> |
| Inputs / Live TV opens incorrectly | TV-specific tuner package or source handler | Identify correct tuner package and update config; avoid disabling input services |
| ADB no longer connects after reboot | TCP debugging may have stopped or TV IP changed | Recheck TV IP, Wi-Fi and Developer options |

## 16. What NOT to do and final security checks

### Avoid these mistakes

- **Do not** blindly copy the <code>examples/oneplus-tv-dubai/debloat.txt</code> list: that is for a **different OnePlus model/firmware**.
- **Do not** disable Google Play Services, Play Store, TV input/tuner, HDMI-CEC, remote, Bluetooth pairing, Wi-Fi, keyboard, firmware update or Chromecast components based on guesses.
- **Do not** run a batch of unknown <code>pm disable-user</code> commands; even a reversible change can lock you out of Wi-Fi, Home or inputs.
- **Do not** use <code>pm uninstall --user 0</code>, <code>adb root</code>, forced firmware commands, or random online APKs as shortcuts.
- **Do not** turn debugging off until you have tested the physical remote, original-launcher recovery plan and a full restart.
- **Do not** share ADB pairing codes, TV settings snapshots or your home's private configuration publicly.
- **Do not** assume an Apple TV-style launcher turns Android TV into Apple's tvOS or removes ads **inside** YouTube and other streaming apps.
- **Do not** assume all Android/Google TVs allow custom Home activities; some manufacturers block sideloading or override the Home key.

### Final checks

- [ ] CleanYourTV appears when Home is pressed.
- [ ] The interface still appears after a full TV restart.
- [ ] Streaming apps work with their original accounts.
- [ ] HDMI inputs and physical remote work.
- [ ] TV and soundbar audio work; if you use ARC, retest ARC after restart.
- [ ] Backup folder remains private on the Mac.
- [ ] You have the stock-launcher recovery commands saved.

**Important network security:** When you are satisfied, on the TV turn **USB debugging / Network debugging / Wireless debugging OFF**, revoke debugging authorizations if that option is available, and restart if needed. TCP ADB port 5555 may be accessible to other devices on your LAN while debugging is active. On your Mac, check the port is closed:

~~~bash
nc -vz -G 3 "$TV_IP" 5555
~~~

This last check should **not** report success. If it remains open, inspect the TV's ADB/debug settings and trusted-network exposure. Keep the backup because turning debugging off makes emergency recovery harder.

---

## Credits, scope and support

This is a **community, step-by-step Mac walkthrough** written for the fork at [anuanurag2410/cleanyourtv](https://github.com/anuanurag2410/cleanyourtv), based on a real OnePlus Y1S Pro setup and its observed troubleshooting. The **CleanYourTV software, launcher and main project documentation originate with [Gathin23/cleanyourtv](https://github.com/Gathin23/cleanyourtv)** and its contributors. Their existing repository, license and attribution remain intact.

For advanced launchers, themes, official device examples and project issues, see:
- [Original project](https://github.com/Gathin23/cleanyourtv)
- [Original setup prompt](../PROMPT.md)
- [Original troubleshooting documentation](../docs/TROUBLESHOOTING.md)
- [OnePlus official specifications](https://www.oneplus.in/tv-y1s-pro)

**Share this guide with your Instagram audience:**  
https://github.com/anuanurag2410/cleanyourtv/blob/main/docs/ONEPLUS_Y1S_PRO_MAC_SETUP.md
