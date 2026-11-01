# Fire TV setup and customisation guide

A reusable guide to Windows ADB, Projectivy configuration, optional wallpapers, appearance settings, troubleshooting, and rollback. Local connection values are requested at runtime.

## 1. Scope and prerequisites

- A Windows computer on the same home network as the TV.
- An Android-based Fire TV that supports network ADB debugging.
- Google's Platform Tools and the official Projectivy release.
- Access to the TV remote to approve debugging and configure the launcher.

Menu names and supported features can vary by app and firmware version. Verify each operation on the device before treating the setup as complete.

## 2. Where commands run

Run the PowerShell commands **on a Windows computer connected to the TV's home network**.

Read the current **IP Address** from the TV's network information. Enter it locally when prompted below. The gateway/router address is a different field. Device addresses can change.

## 3. Install and locate ADB on Windows

Install Google's Platform Tools:

~~~powershell
winget install --id Google.PlatformTools --exact --source winget
~~~

Restart PowerShell/Windows Terminal after installation. If `adb` is still unrecognised, locate the executable directly:

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

These optional read-only checks identify the TV and its starting configuration:

~~~powershell
& $adbPath -s $tvDevice shell getprop ro.product.model
& $adbPath -s $tvDevice shell getprop ro.build.version.release
& $adbPath -s $tvDevice shell cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME
& $adbPath -s $tvDevice shell df -h /data
~~~

The storage command reports the current data-partition usage.

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

Use the default local ADB server unless a specific diagnostic requires a different configuration.

## 5. Install and open Projectivy

Official release source: <https://github.com/spocky/miproja1/releases>.

For a first installation, fetch the current release metadata, select its actual APK asset, and check its size and any published SHA-256 digest:

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

### Verify the active Home handler

An Amazon HOME activity can look like:

~~~text
com.amazon.tv.launcher/.ui.HomeActivity_vNext
~~~

On some firmware, `cmd package set-home-activity` may report `Success` while HOME still resolves to Amazon. Check the resolved activity and physical Home-button behaviour rather than relying on the exit code alone. The accessibility-based procedure below keeps Amazon's launcher available.

### Diagnose accessibility

Check whether accessibility is enabled and which services are configured and running:

~~~powershell
& $adbPath -s $tvDevice shell settings get secure accessibility_enabled
& $adbPath -s $tvDevice shell settings get secure enabled_accessibility_services
& $adbPath -s $tvDevice shell dumpsys accessibility
~~~

The normal TV menu is **Fire TV Home → Settings → Accessibility**. It can show built-in options such as Subtitle, Alexa Subtitles, VoiceView, Text Banner, and Screen Magnifier. Availability of third-party services varies.

Identify the installed Projectivy service with:

~~~powershell
& $adbPath -s $tvDevice shell cmd package query-services --brief -a android.accessibilityservice.AccessibilityService -p com.spocky.projengmenu
~~~

The expected component is:

~~~text
com.spocky.projengmenu/.services.ProjectivyAccessibilityService
~~~

### Enable the identified service with a backup

This grants Projectivy accessibility access to interface events needed for its launcher override.

The following procedure is for **a state with accessibility disabled and no enabled services**. Its guard stops if other settings are present. If Projectivy is already enabled and bound, skip this block and verify its status instead.

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

An enabled and bound service can report:

~~~text
1
Bound services:{Service[label=Projectivy Launcher, ...]}
Enabled services:{{com.spocky.projengmenu/com.spocky.projengmenu.services.ProjectivyAccessibilityService}}
~~~

The **Bound services** entry is runtime evidence. A configured service name alone does not establish that it is running.

If a write fails partway through, inspect the current values and use the saved backup to restore them before another attempt; the block does not automatically roll back partial changes.

### Turn on the app's override and verify

1. Open Projectivy directly.
2. Select **gear → Projectivy settings → General**.
3. Enable **Override current launcher**.
4. Press the **physical remote's Home button**, wait about three seconds, and observe which launcher remains.

A screenshot showing Projectivy open does not establish that the Home button or startup behaviour works. Test the physical Home button and a full TV restart before treating the override as persistent. A brief Amazon screen may occur with a window-event-based override.

## 7. Free appearance settings

The preset below uses compact cards, an existing app order, and a visible Mobile Apps category. Adjust it to suit the desired layout.

Under **Projectivy settings → Appearance**, look for **Categories / Channels**, **Cards**, **Wallpaper**, **Status Bar**, **Left headers**, **Launcher Idle Mode**, **Font**, and **Themes**.

For a dark background without a rotating image source, select **Wallpaper → Launcher wallpaper → None** and check the result on the main launcher.

Suggested settings for a simpler row presentation:

| Location | Setting | Suggested value |
| --- | --- | --- |
| Cards | Show title | On, to retain the focused app's name |
| Cards | Rounded corners | Around one-third of the slider |
| Cards | Cards size | Choose a comfortable small size |
| Categories / Channels | Show title | Off |
| Categories / Channels | Show icon in front of the title | Off |
| Categories / Channels | Vertical spacing | Low; one or two right presses from minimum |
| Categories / Channels | Keep focused category position fixed | On |
| Categories / Channels | Hide rows above selection | Off |

The two **Show title** controls affect different things: the app name versus category headings. Hiding a heading does not hide the category itself.

Use the gear's **Edit categories** menu to control category visibility. Keep Mobile Apps visible when that row is wanted.

## 8. Optional glass-inspired preset

The Cards colour picker can expose **Luminance**, **Opacity**, and **Prefer dynamic colour**, with a separate **Focused card** submenu.

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

These are translucent backing and soft-highlight settings, not a guarantee of true frosted backdrop blur or iOS Liquid Glass refraction. Opaque app artwork may cover its card backing.

Verify the result on the main launcher: press **Back** until settings panels close and the app cards appear, then inspect focused and unfocused cards.

## 9. Wallpapers and Premium features

Wallpaper-source options can include:

- None
- Reddit
- Wikimedia Commons
- Dynamic colours
- Default wallpaper
- **Media file**, marked Premium

**Themes** and **Icon pack** can also require Premium. Features marked Premium require a valid unlock. Resolve checkout or billing problems through the provider; the free appearance settings above remain available independently.

An optional generated blue wallpaper is bundled here:

![Original Apple-style blue wallpaper](assets/apple-style-wallpaper.png)

[Open the PNG](assets/apple-style-wallpaper.png).

Alternative macOS wallpapers are available through the [512 Pixels macOS wallpaper archive](https://512pixels.net/projects/default-mac-wallpapers-in-5k/). Download, TV transfer, and application are separate operations to verify.

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

This guide does not establish a free custom-image route through **Default wallpaper**.

### File and PowerShell troubleshooting

- **`cannot stat ... No such file or directory`:** the Windows source path is wrong or the file has not been saved. Inspect the actual image path; do not assume a Desktop filename exists.
- **An image preview is not a saved Windows file.** Save it first, or use the asset from a local checkout.
- **File picker seems to do nothing:** look behind PowerShell with **Alt+Tab**. A modal file picker pauses the command until a file is selected or the dialog is cancelled.
- **`>>` continuation prompt:** PowerShell may be waiting for the rest of a block. Close a missing `}` only when that is what the unfinished block requires; Ctrl+C cancels incomplete input.
- **Recent files lists:** confirm the selected file is the wallpaper rather than a TV screenshot. A top-ten list is not proof that no wallpaper exists elsewhere or with an older timestamp.

## 10. Capture a TV screenshot for verification

Use a temporary file on the TV and pull it to Windows. This avoids binary screenshot data passing through Windows PowerShell text redirection.

Keep device screenshots and settings backups local; neither is needed in the published guide.

~~~powershell
$name = 'Projectivy-' + (Get-Date -Format 'yyyyMMdd-HHmmss') + '.png'
$shot = Join-Path ([Environment]::GetFolderPath('Desktop')) $name
& $adbPath -s $tvDevice shell screencap -p ('/sdcard/Download/' + $name)
& $adbPath -s $tvDevice pull ('/sdcard/Download/' + $name) $shot
Invoke-Item $shot
~~~

An `am start` warning saying the intent was delivered to a currently running top-most instance means that activity was already open. It does not confirm that a permission was enabled.

## 11. Restore a saved accessibility state

The enablement procedure writes a file named `TV-accessibility-backup-<GUID>.json` on the Windows Desktop. Use the locally saved backup.

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

Restoring a disabled accessibility state stops Projectivy's accessibility-based Home override. If a preferred-home selection was changed separately, restore the exact original component recorded before that change:

~~~powershell
$originalHome = Read-Host 'Enter the original HOME component recorded before changing it'
if ($originalHome -notmatch '^[A-Za-z0-9_.]+/[A-Za-z0-9_.$]+$') {
    throw 'Invalid HOME component.'
}
& $adbPath -s $tvDevice shell cmd package set-home-activity --user $androidUser ("'" + $originalHome + "'")
~~~

## 12. Verification checklist

- Verify the glass-inspired settings on the **main launcher**, outside the settings preview.
- Confirm the physical **Home** button returns to Projectivy.
- After configuration is satisfactory, perform a full TV restart and repeat the Home test before describing the setup as persistent.
- Confirm that a representative streaming app opens and returns to the launcher.
- Custom wallpaper application remains dependent on a usable Media file unlock or another independently verified supported route.
- Assess responsiveness separately. Appearance changes alone do not establish faster app loading or processor performance.

The command blocks form a reference workflow, not a device-specific execution log. Validate them on the intended TV; the complete workflow and rollback have not been tested across all firmware versions.

This guide and its bundled image are documentation assets. They do not apply settings to the TV or publish repository changes.
