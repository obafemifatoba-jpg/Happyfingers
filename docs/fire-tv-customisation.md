# TCL Fire TV setup and customisation guide

This records the necessary steps from the TV customisation session on **1 November 2026**. It separates changes demonstrated on the TV from suggested settings and checks that are still outstanding.

## 1. Device, tools, and recorded outcome

| Item | Recorded value or outcome |
| --- | --- |
| TV | TCL 43PF650K |
| Model reported by ADB | `AFTMA475B1` |
| Fire OS | 8.1.1.5, reported by the owner |
| Underlying Android | 11, confirmed through ADB |
| Computer | Windows, using PowerShell |
| Platform Tools | 37.0.1; ADB 1.0.41 was reported |
| Connection | Windows-to-TV network ADB authenticated successfully |
| Launcher | Projectivy installed from its developer's GitHub releases and opened successfully |
| Accessibility | Projectivy's accessibility service was confirmed enabled and bound/running |
| Amazon launcher | Retained; Android's HOME resolution still reported Amazon after a default-home change attempt |
| Appearance | Dark launcher background, small cards, and hidden category headings were visible in screenshots |
| Final preferences | Keep smaller cards, the existing main app order, and Mobile Apps visible |
| Glass styling | Transparency and glow controls were found; the suggested preset has not yet been verified on the main launcher |
| Wallpaper | Original PNG prepared; owner reported the later macOS wallpaper download completed. Successful application was not confirmed |
| Premium | Media file wallpaper import, Themes, and Icon pack showed Premium markers. The owner reported a payment-provider error |
| Speed | No system animation-scale changes or performance benchmarks were performed |

The exact Projectivy version was not recorded. Menu names below follow the screens actually shown during the session.

## 2. Where commands run

Run the PowerShell commands **on the Windows computer connected to the TV's home network**. The cloud chat did not have direct access to the home LAN; the owner executed commands locally and shared results.

The TV used `192.168.0.102` during the session. Its address can change, so read the current **IP Address** from the TV's network information. The gateway/router address is a different field; `192.168.0.1` was the initial mistaken address.

## 3. Install and locate ADB on Windows

Install Google's Platform Tools:

~~~powershell
winget install --id Google.PlatformTools --exact --source winget
~~~

Restart PowerShell/Windows Terminal after installation. If `adb` is still unrecognised, locate the executable directly; this was the working solution in the session:

~~~powershell
$folders = @(
    (Join-Path $env:LOCALAPPDATA 'Microsoft\WinGet\Packages')
    (Join-Path $env:ProgramFiles 'WinGet\Packages')
)

$adbPath = Get-ChildItem -Path $folders -Filter adb.exe -File -Recurse -ErrorAction SilentlyContinue |
    Where-Object { $_.FullName -match '[\\/]Google\.PlatformTools_' } |
    Select-Object -First 1 -ExpandProperty FullName

if (!$adbPath) { throw 'Google Platform Tools adb.exe was not found.' }
& $adbPath version
~~~

Keep using `& $adbPath` in this PowerShell session. A new window needs these variables set again.

## 4. Enable debugging and connect

On the TV:

1. Open **Settings → Device & Software → About**.
2. Highlight the device name and press **Select seven times** to reveal Developer Options, if necessary.
3. Open **Developer Options → ADB Debugging** and enable it.
4. Read the TV's **IP Address** under **About → Network**.

On Windows:

~~~powershell
$tvAddress = Read-Host 'Enter the TV IP address'
$tvDevice = '{0}:5555' -f $tvAddress
& $adbPath connect $tvDevice
& $adbPath devices -l
& $adbPath -s $tvDevice get-state
~~~

Accept the debugging authorisation prompt **on the TV**. Select **Always allow from this computer** if offered and appropriate for this PC. A successful state is `device`; `unauthorized` means authorisation is incomplete.

The read-only checks used to identify the TV and its starting configuration were:

~~~powershell
& $adbPath -s $tvDevice shell getprop ro.product.model
& $adbPath -s $tvDevice shell getprop ro.build.version.release
& $adbPath -s $tvDevice shell cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME
& $adbPath -s $tvDevice shell pm list packages -3
& $adbPath -s $tvDevice shell df -h /data
~~~

Storage at that point was approximately **8.7 GB total, 6.3 GB used, 2.2 GB available**. This was a point-in-time observation.

### Connection troubleshooting

- **ADB not recognised:** use the executable discovery block above; reinstalling was unnecessary.
- **Connection appears to hang:** press **Ctrl+C**, then check the port:

  ~~~powershell
  Test-NetConnection -ComputerName $tvAddress -Port 5555
  ~~~

  `TcpTestSucceeded : True` establishes port reachability, not ADB authorisation.
- **Authentication fails:** keep the TV awake, toggle its ADB Debugging off and on, reconnect, and accept the new TV prompt. Run commands one at a time to identify a stall.
- **Restarting ADB if needed:** the commands below restart the PC's ADB server and interrupt its existing ADB connections:

  ~~~powershell
  & $adbPath kill-server
  & $adbPath start-server
  & $adbPath connect $tvDevice
  & $adbPath devices -l
  ~~~

An alternate PC server port and Windows debugging-key regeneration were suggested during troubleshooting, but neither was established as necessary. The owner reported that the original/default-server connection worked. They are not part of this guide's normal procedure.

## 5. Install and open Projectivy

Official release source: <https://github.com/spocky/miproja1/releases>.

The session used the current release metadata, selected the actual APK asset, checked its size and any published SHA-256 digest, and installed it. For a first installation:

~~~powershell
& {
    $ErrorActionPreference = 'Stop'
    $release = Invoke-RestMethod -Uri 'https://api.github.com/repos/spocky/miproja1/releases/latest' -Headers @{ 'User-Agent' = 'TV-Setup' }
    $assets = @($release.assets | Where-Object { $_.name -like '*.apk' })
    if ($assets.Count -ne 1) {
        $assets | Select-Object name, browser_download_url | Format-List
        throw 'Select the appropriate official APK before continuing.'
    }

    $asset = $assets[0]
    if ($asset.browser_download_url -notlike 'https://github.com/spocky/miproja1/releases/download/*') {
        throw 'Unexpected download source.'
    }
    $apk = Join-Path $env:TEMP ('Projectivy-' + [guid]::NewGuid() + '.apk')
    Invoke-WebRequest -UseBasicParsing -Uri $asset.browser_download_url -OutFile $apk
    if ((Get-Item $apk).Length -ne $asset.size) { throw 'Download size mismatch.' }

    if ($asset.digest) {
        if ($asset.digest -notmatch '^sha256:([a-fA-F0-9]{64})$') {
            throw 'Unsupported checksum format.'
        }
        $expectedHash = $Matches[1]
        if ((Get-FileHash $apk -Algorithm SHA256).Hash -ne $expectedHash) {
            throw 'Checksum mismatch.'
        }
    }

    & $adbPath -s $tvDevice install $apk
    if ($LASTEXITCODE -ne 0) { throw 'Installation failed; inspect the output.' }
}
~~~

Skip installation when Projectivy is already installed. An absent release digest means there was no published checksum to compare; computing a hash alone does not independently verify an APK's publisher.

Open Projectivy directly:

~~~powershell
& $adbPath -s $tvDevice shell am start -W -a android.intent.action.MAIN -c android.intent.category.LAUNCHER -p com.spocky.projengmenu
~~~

Use the highlighted bottom arrow to finish the welcome pages.

**Navigation:** the gear opens a quick menu. Select **Projectivy settings** in that menu to reach the full settings page. Its relevant sections are **General**, **Appearance**, and **Accessibility service**. The actual appearance menu is named **Appearance**, not “User interface.”

## 6. Configure Home-button handling

### Default-home change did not take effect

The original Android HOME activity was:

~~~text
com.amazon.tv.launcher/.ui.HomeActivity_vNext
~~~

An attempt to select Projectivy using `cmd package set-home-activity` returned `Success` and exit code `0`. Immediately resolving HOME still returned Amazon. That attempt did **not** establish a working Home replacement.

A command to restore the preferred-home selection to the recorded Amazon activity was provided. Its execution was not separately confirmed. Amazon's launcher was kept installed and enabled.

### Diagnose accessibility

The first accessibility checks returned `0`, an empty enabled-services value, and no bound services:

~~~powershell
& $adbPath -s $tvDevice shell settings get secure accessibility_enabled
& $adbPath -s $tvDevice shell settings get secure enabled_accessibility_services
& $adbPath -s $tvDevice shell dumpsys accessibility
~~~

The normal TV menu is **Fire TV Home → Settings → Accessibility**. The screenshot showed built-in options such as Subtitle, Alexa Subtitles, VoiceView, Text Banner, and Screen Magnifier. No successful Projectivy enablement through that menu was recorded.

The installed service was identified with:

~~~powershell
& $adbPath -s $tvDevice shell cmd package query-services --brief -a android.accessibilityservice.AccessibilityService -p com.spocky.projengmenu
~~~

It returned:

~~~text
com.spocky.projengmenu/.services.ProjectivyAccessibilityService
~~~

### Enable the identified service with a backup

This grants Projectivy accessibility access to interface events needed for its launcher override.

The following procedure was executed successfully **from a state with accessibility disabled and no enabled services**. Its guard intentionally stops if that state has changed. If Projectivy is already enabled and bound, skip this block and verify its status instead.

~~~powershell
$androidUser = (& $adbPath -s $tvDevice shell am get-current-user).Trim()

& {
    $ErrorActionPreference = 'Stop'
    if ($androidUser -notmatch '^\d+$') { throw 'Could not identify the Android user.' }
    $tvSettingsArgs = @('shell', 'settings', '--user', $androidUser)

    $oldEnabled = ((& $adbPath -s $tvDevice @tvSettingsArgs get secure accessibility_enabled) -join '').Trim()
    $oldServices = ((& $adbPath -s $tvDevice @tvSettingsArgs get secure enabled_accessibility_services) -join '').Trim()
    if ($oldEnabled -ne '0' -or $oldServices -notin @('', 'null')) {
        throw 'Existing accessibility settings differ. Preserve them before proceeding.'
    }

    $backup = Join-Path ([Environment]::GetFolderPath('Desktop')) ('TV-accessibility-backup-' + [guid]::NewGuid() + '.json')
    @{
        Device = $tvDevice
        AndroidUser = $androidUser
        AccessibilityEnabled = $oldEnabled
        EnabledServices = $oldServices
    } | ConvertTo-Json | Out-File -FilePath $backup -Encoding UTF8 -NoClobber
    Write-Host ('Backup saved: ' + $backup)

    $service = 'com.spocky.projengmenu/com.spocky.projengmenu.services.ProjectivyAccessibilityService'
    & $adbPath -s $tvDevice @tvSettingsArgs put secure enabled_accessibility_services $service
    if ($LASTEXITCODE -ne 0) { throw 'Service setting failed; inspect the output.' }

    $actual = ((& $adbPath -s $tvDevice @tvSettingsArgs get secure enabled_accessibility_services) -join '').Trim()
    if ($actual -notin @($service, 'com.spocky.projengmenu/.services.ProjectivyAccessibilityService')) {
        throw 'The service setting was not retained. Inspect the state before continuing.'
    }

    & $adbPath -s $tvDevice @tvSettingsArgs put secure accessibility_enabled 1
    if ($LASTEXITCODE -ne 0) { throw 'Accessibility could not be enabled; inspect the state.' }

    Start-Sleep -Seconds 2
    & $adbPath -s $tvDevice @tvSettingsArgs get secure accessibility_enabled
    & $adbPath -s $tvDevice shell dumpsys accessibility |
        Select-String -Pattern 'Bound services|com\.spocky\.projengmenu|denied|denial|exception'
}
~~~

The owner supplied output showing:

~~~text
1
Bound services:{Service[label=Projectivy Launcher, ...]}
Enabled services:{{com.spocky.projengmenu/com.spocky.projengmenu.services.ProjectivyAccessibilityService}}
~~~

That confirms the service was running. A configured service name alone would not establish that.

If a write fails partway through, inspect the current values and use the saved backup to restore them before another attempt; the block does not automatically roll back partial changes.

### Turn on the app's override and verify

1. Open Projectivy directly.
2. Select **gear → Projectivy settings → General**.
3. Enable **Override current launcher**.
4. Press the **physical remote's Home button**, wait about three seconds, and observe which launcher remains.

A screenshot showing Projectivy open does not establish that the Home button or startup behaviour works. Explicit Home-button and full-TV-restart confirmation remain outstanding in the session record. A brief Amazon screen may occur with a window-event-based override.

## 7. Final layout preferences and free appearance settings

The owner's final preferences take precedence over earlier suggestions:

- **Keep cards small** at the chosen size.
- **Keep the existing main app order**.
- **Keep Mobile Apps visible**.

Earlier suggestions to enlarge cards, reorder favourites, or hide Mobile Apps were superseded.

Under **Projectivy settings → Appearance**, the confirmed sections included **Categories / Channels**, **Cards**, **Wallpaper**, **Status Bar**, **Left headers**, **Launcher Idle Mode**, **Font**, and **Themes**.

The free dark-background route used **Wallpaper → Launcher wallpaper → None**. Later screenshots showed a dark background without the earlier rotating scenery.

For the simpler row presentation, the suggested settings were:

| Location | Setting | Suggested value |
| --- | --- | --- |
| Cards | Show title | On, to retain the focused app's name |
| Cards | Rounded corners | Around one-third of the slider |
| Cards | Cards size | Owner's preferred small size |
| Categories / Channels | Show title | Off |
| Categories / Channels | Show icon in front of the title | Off |
| Categories / Channels | Vertical spacing | Low; one or two right presses from minimum |
| Categories / Channels | Keep focused category position fixed | On |
| Categories / Channels | Hide rows above selection | Off |

The two **Show title** controls affect different things: the app name versus category headings. Hiding a heading does not hide the category itself.

The gear's **Edit categories** menu was inspected. Leave its Mobile Apps category visible, as subsequently requested.

## 8. Glass-inspired preset: suggested, not yet verified

The owner then requested an iOS-style glass look. The Cards screens confirmed a colour picker with **Luminance**, **Opacity**, and **Prefer dynamic colour**, plus a **Focused card** submenu.

Suggested settings under **Appearance → Cards**:

| Control | Suggested value |
| --- | --- |
| Ripple effect on card press | Off |
| Background colour | Grey swatch immediately left of the X |
| Background colour: Prefer dynamic colour | Off |
| Background colour: Luminance | 0 |
| Background colour: Opacity | About one-third along the track |
| Colour picker confirmation | Select OK |
| Focused card effect | Glow |
| Glow animation | None, if the dropdown offers it |
| Glow size | Approximately 10–15% of the track |
| Glow colour | Grey swatch with luminance raised toward white |
| Glow colour: Opacity | Around halfway along the track |
| Glow colour: Prefer dynamic colour | Off; confirm OK |

These are translucent backing and soft-highlight settings. The controls shown did not expose true frosted backdrop blur or iOS Liquid Glass refraction. Opaque app artwork may cover its card backing.

The last screenshot after this advice showed the root **Projectivy Launcher settings** page, not the main app view. The final glass-inspired appearance is therefore **not verified**.

To check it, press **Back** until settings panels close and the app cards appear, then inspect the focused and unfocused cards.

## 9. Wallpapers and the Premium/payment limitation

The actual wallpaper-source menu showed:

- None
- Reddit
- Wikimedia Commons
- Dynamic colours
- Default wallpaper
- **Media file**, marked Premium

**Themes** and **Icon pack** also showed Premium markers. The owner reported that the licence purchase failed with a temporary payment-provider issue. No successful purchase or Premium activation was established.

The original generated blue wallpaper is bundled here:

![Original Apple-style blue wallpaper](assets/apple-style-wallpaper.png)

[Open the PNG](assets/apple-style-wallpaper.png).

A macOS Big Sur night wallpaper was later obtained through the [512 Pixels macOS wallpaper archive](https://512pixels.net/projects/default-mac-wallpapers-in-5k/). The owner confirmed the download, but did not supply independent confirmation that it was copied to the TV or applied.

### Copy a known local image to the TV

From a **local Windows checkout of this repository**, use the bundled PNG:

~~~powershell
$wallpaper = (Resolve-Path -LiteralPath '.\docs\assets\apple-style-wallpaper.png' -ErrorAction Stop).Path
~~~

For an image saved elsewhere, read its actual path instead:

~~~powershell
$wallpaper = (Read-Host 'Full path to the wallpaper image').Trim('"')
if (!(Test-Path -LiteralPath $wallpaper -PathType Leaf)) {
    throw 'Image file not found. Locate the saved image before continuing.'
}
~~~

Then copy to a new filename:

~~~powershell
& {
    $extension = [IO.Path]::GetExtension($wallpaper)
    if ($extension -notin @('.png', '.jpg', '.jpeg', '.webp')) {
        throw 'Select a supported image file.'
    }
    $tvWallpaper = '/sdcard/Pictures/TVStyle/wallpaper-' + [guid]::NewGuid().ToString('N') + $extension
    & $adbPath -s $tvDevice shell mkdir -p /sdcard/Pictures/TVStyle
    if ($LASTEXITCODE -ne 0) { throw 'Could not create the TV folder.' }
    & $adbPath -s $tvDevice push $wallpaper $tvWallpaper
    if ($LASTEXITCODE -ne 0) { throw 'Wallpaper transfer failed.' }
    Write-Host ('Wallpaper copied to: ' + $tvWallpaper)
}
~~~

A successful download, a successful `adb push`, and a successfully applied wallpaper are **three separate outcomes**. The upload should report `1 file pushed`.

When Media file is unlocked, select **Launcher wallpaper → Media file**, open **Configure** if offered, and select the image from **Pictures → TVStyle**. A Premium checkout prompt means this application route remains blocked.

No free custom-image route through **Default wallpaper** was verified. No payment or licence bypass was performed.

### File and PowerShell troubleshooting

- **`cannot stat ... No such file or directory`:** the Windows source path is wrong or the file has not been saved. Inspect the actual image path; do not assume a Desktop filename exists.
- **A wallpaper shown in chat is not a downloaded Windows file.** Save it first, or use the asset from a local checkout.
- **File picker seems to do nothing:** look behind PowerShell with **Alt+Tab**. A modal file picker pauses the command until a file is selected or the dialog is cancelled.
- **`>>` continuation prompt:** PowerShell may be waiting for the rest of a block. Close a missing `}` only when that is what the unfinished block requires; Ctrl+C cancels incomplete input.
- **Recent files list only Projectivy images:** those are the TV screenshots. A top-ten list is not proof that no wallpaper exists elsewhere or with an older timestamp.

## 10. Capture a TV screenshot for verification

Use a temporary file on the TV and pull it to Windows. This avoids binary screenshot data passing through Windows PowerShell text redirection.

~~~powershell
$name = 'Projectivy-' + (Get-Date -Format 'yyyyMMdd-HHmmss') + '.png'
$shot = Join-Path ([Environment]::GetFolderPath('Desktop')) $name
& $adbPath -s $tvDevice shell screencap -p ('/sdcard/Download/' + $name)
& $adbPath -s $tvDevice pull ('/sdcard/Download/' + $name) $shot
Invoke-Item $shot
~~~

An `am start` warning saying the intent was delivered to a currently running top-most instance means that activity was already open. It does not confirm that a permission was enabled.

## 11. Restore the recorded accessibility state

The enablement procedure saved a file named `TV-accessibility-backup-<GUID>.json` on the Windows Desktop. That file is on the owner's PC, not in this repository.

Use the backup for this TV and the active local connection. The restoration order is enabled services first, then the global accessibility flag:

~~~powershell
& {
    $ErrorActionPreference = 'Stop'
    $backupFile = (Read-Host 'Full path to the TV accessibility backup JSON').Trim('"')
    $before = Get-Content -LiteralPath $backupFile -Raw | ConvertFrom-Json
    $userId = [string]$before.AndroidUser
    $services = [string]$before.EnabledServices
    $enabled = [string]$before.AccessibilityEnabled

    if ($userId -notmatch '^\d+$' -or $enabled -notin @('0', '1', 'null') -or
        ($services -ne 'null' -and $services -notmatch '^[A-Za-z0-9_.$/:]*$')) {
        throw 'Unexpected backup values. Inspect the backup before restoring.'
    }

    $tvSettingsArgs = @('shell', 'settings', '--user', $userId)
    foreach ($entry in @(
        @{ Name = 'enabled_accessibility_services'; Value = $services }
        @{ Name = 'accessibility_enabled'; Value = $enabled }
    )) {
        if ($entry.Value -eq 'null') {
            & $adbPath -s $tvDevice @tvSettingsArgs delete secure $entry.Name
        } else {
            # Single quotes preserve the complete remote argument, including an empty value.
            $quotedValue = "'" + $entry.Value + "'"
            & $adbPath -s $tvDevice @tvSettingsArgs put secure $entry.Name $quotedValue
        }
        if ($LASTEXITCODE -ne 0) { throw ('Restore failed for ' + $entry.Name) }
        $actual = ((& $adbPath -s $tvDevice @tvSettingsArgs get secure $entry.Name) -join '').Trim()
        if ($LASTEXITCODE -ne 0 -or $actual -cne $entry.Value) {
            throw ('Restore verification failed for ' + $entry.Name)
        }
        Write-Host ($entry.Name + ': restored ' + $actual)
    }
}
~~~

Check the displayed values against the backup. A literal `null` denotes an absent setting and is restored with `settings delete`; it is not written as a string value. Empty values are passed as an explicitly quoted empty argument.

Restoring the original disabled accessibility state stops Projectivy's accessibility-based Home override. To restore the recorded Amazon preferred-home selection if needed:

~~~powershell
& $adbPath -s $tvDevice shell cmd package set-home-activity --user $androidUser com.amazon.tv.launcher/.ui.HomeActivity_vNext
~~~

## 12. Remaining checks

- Verify the glass-inspired settings on the **main launcher**, outside the settings preview.
- Confirm the physical **Home** button returns to Projectivy.
- After configuration is satisfactory, perform a full TV restart and repeat the Home test before describing the setup as persistent.
- Confirm that a representative streaming app opens and returns to the launcher.
- Custom wallpaper application remains dependent on a usable Media file unlock or another independently verified supported route.
- Responsiveness changes and speed measurements remain outstanding. No system animation-scale changes were applied, and app-loading or processor speed was not measured.

The command blocks consolidate the session into a reference workflow. The entire guide has not been rerun from a fresh Windows/TV setup, and the rollback block has not been exercised. Individual confirmed outcomes are recorded above.

This guide and its bundled image are documentation assets. They do not apply settings to the TV or publish repository changes.
