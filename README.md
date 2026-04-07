# Mac OSX Wi-Fi Location Changer

English | [简体中文](README.zh-CN.md)

- Automatically changes the macOS network location when a configured Wi-Fi (SSID) becomes connected
- Allows having different IP settings depending on the Wi-Fi SSID
- Offers a hook to run an external script when the location changes

## Read this first on macOS 15.7.5 / Sequoia

If you're using macOS Sequoia, especially 15.6+ or 15.7.x, read this section before installing or troubleshooting.

- **Shortcuts-based SSID detection is now the primary path on macOS 15+**. The scripts use `shortcuts run ... --output-type public.plain-text --output-path ...` instead of relying on stdout only, which is more reliable under `launchd`.
- **Shortcut name compatibility was improved**. The scripts accept both `Current Wi-Fi` and the older `Current WiFi` name.
- **Additional best-effort fallback methods were added**. If Shortcuts is unavailable, the scripts now try `ipconfig getsummary`, `networksetup -getairportnetwork`, preferred network list inference, `defaults read`, and `system_profiler` in that order.
- **The configuration UI now handles macOS 15 correctly**. It no longer treats Sequoia as macOS 26, reports mapping counts correctly, prints available locations more clearly, and avoids saving placeholder text like `<Unable to detect - Privacy Protected>` as an SSID mapping.
- **The LaunchAgent plist was cleaned up**. Invalid `WatchPaths` entries that were not actual preference files were removed.
- **No compiled binaries were changed or required**. This project currently consists of shell scripts plus a LaunchAgent plist.

Important behavior note:

- Running `bash ./locationchanger` from the repository uses the repository-local config path (`./locationchanger.conf`) but still expects the privileged helper at `/usr/local/bin/locationchanger-helper`.
- For real verification, install the project first with `./install.sh`, ensure the sudoers entry is present, and then test `/usr/local/bin/locationchanger` or the loaded LaunchAgent.

Known limitation on newer Sequoia builds:

- On some macOS 15.6+ / 15.7.x systems, Apple appears to restrict shell-only SSID access more aggressively. In those cases, the most reliable short-term solution is to use the Shortcuts app to expose the current SSID.
- If Shortcuts works but shell fallbacks do not, the project can still function after installation.
- If neither Shortcuts nor shell fallbacks can read the SSID, a pure script-based solution may no longer be sufficient on that machine; a signed app/agent approach would be the long-term fix.

## Quick start

### Install and configure LocationChanger

1. If you're on macOS 15+ / Sequoia, create and test the `Current Wi-Fi` shortcut first:

   ```bash
   tmp=$(mktemp)
   shortcuts run "Current Wi-Fi" --output-type public.plain-text --output-path "$tmp"
   cat "$tmp"
   rm -f "$tmp"
   ```

   If you already created `Current WiFi`, the scripts accept that legacy name too.

2. Install LocationChanger:

   ```bash
   ./install.sh
   ```

3. Configure SSID mappings:

   ```bash
   ./config-ui.sh
   ```

   - Select option 1 to add SSID mappings
   - Select option 5 to install if not already done

4. Set up sudoers. This is required for location switching:

   ```bash
   sudo visudo
   ```

   Add this line and replace `your_username` with your actual username:

   ```text
   your_username ALL=(ALL) NOPASSWD: /usr/local/bin/locationchanger-helper
   ```

5. Test the installed service by switching between different Wi-Fi networks.

## Configure LocationChanger

The easiest way to configure LocationChanger is with the interactive configuration tool:

```bash
./config-ui.sh
```

This gives you a command-line UI for the full setup flow. Check out [Use the configuration UI](#use-the-configuration-ui) for the details.

### macOS notifications

The script triggers a macOS notification after the location changes. If you don't want that behavior, delete the lines that start with `osascript`.

## Install LocationChanger

### Recommended: use the configuration tool

The easiest way to install LocationChanger is through the configuration tool:

```bash
./config-ui.sh
```

Then select option 5, **Install LocationChanger**. This will:

- Run the installation script
- Preserve any existing configurations
- Provide guidance for the next steps

### Install automatically

Run:

```bash
./install.sh
```

### Install manually

Copy these files:

```bash
cp locationchanger /usr/local/bin
cp locationchanger-helper /usr/local/bin
cp locationchanger.conf /usr/local/bin
cp LocationChanger.plist ~/Library/LaunchAgents/
```

Make the scripts executable:

```bash
chmod +x /usr/local/bin/locationchanger
chmod +x /usr/local/bin/locationchanger-helper
sudo chown root /usr/local/bin/locationchanger-helper
sudo chmod 500 /usr/local/bin/locationchanger-helper
```

Load `LocationChanger.plist` as a launchd daemon:

```bash
launchctl load ~/Library/LaunchAgents/LocationChanger.plist
```

If you place the `locationchanger` script in another location, update the path in `LocationChanger.plist` too.

## Check the logfile

You can adjust the logfile location in `locationchanger`, around line 12:

```bash
exec &>/usr/local/var/log/locationchanger.log
```

Watch the log:

```bash
tail -f /usr/local/var/log/locationchanger.log
```

## Run an external script after the location changes

By convention, placing an executable script in this directory with this name:

`locationchanger.callout.sh`

and then running the installer will cause the LocationChanger service to run that script each time the location changes.

## Test LocationChanger

For testing, configure two locations in your current environment, for example `home` and `guest`, each associated with a different SSID such as your main SSID and guest SSID. Then use the Wi-Fi menu to switch between those SSIDs.

You can inspect any success or error messages written to the log with:

```bash
tail /usr/local/var/log/locationchanger.log
```

## Uninstall LocationChanger

### Recommended: use the configuration tool

```bash
./config-ui.sh
```

Select option 6, **Uninstall LocationChanger**, for a complete removal of all files and configurations.

### Uninstall manually

```bash
# Stop the service
launchctl unload ~/Library/LaunchAgents/LocationChanger.plist

# Remove files
sudo rm -f /usr/local/bin/locationchanger
sudo rm -f /usr/local/bin/locationchanger-helper
sudo rm -f /usr/local/bin/locationchanger.conf
sudo rm -f /usr/local/bin/locationchanger.conf.backup
sudo rm -f /usr/local/bin/locationchanger.callout.sh
rm -f ~/Library/LaunchAgents/LocationChanger.plist
sudo rm -f /usr/local/var/log/locationchanger.log

# Remove sudoers entry manually
sudo visudo
# Remove the line: your_username ALL=(ALL) NOPASSWD: /usr/local/bin/locationchanger-helper
```

## macOS compatibility and security

### Use the Shortcuts app on macOS Sequoia 15.0+

This version supports macOS Sequoia 15.0+ by using the Shortcuts app to detect Wi-Fi SSIDs when traditional methods fail. The script automatically falls back to legacy methods if Shortcuts is not available.

If you see `Privacy Protected` or `<redacted>` in the configuration tool, you can still configure SSID mappings manually. For automatic detection, create a Shortcuts app shortcut named `Current Wi-Fi`. The current script also accepts the legacy name `Current WiFi`.

### Create the `Current Wi-Fi` shortcut

Open the **Shortcuts** app. Click **+** to add a new shortcut. In the right pane, select **Device** and drag **Get Network Details** into the shortcut. Then go to **Scripting**, find **Stop and Output**, and place it below the first action. Run it with the **Play** button and confirm it shows your current Wi-Fi name. Save the shortcut as **Current Wi-Fi**.

If you already created **Current WiFi**, the script accepts that legacy name too.

It should look like this:

<img width="2446" height="1624" alt="Shortcuts App New shortcut 3" src="https://github.com/user-attachments/assets/9eae481c-4881-478b-a9a2-86c0363a955e" />

### Use secure location switching

Starting with macOS Sequoia 15.5+, changing network locations requires admin privileges. This project uses a secure helper script approach to minimize security risks.

Security benefits:

- Only the specific helper script can be executed without a password
- The helper script is owned by root with restricted permissions (`500`)
- The helper script only allows network location switching, which helps prevent privilege escalation
- Input validation helps prevent injection attacks

### Use the automatic fallback location

If no SSID mapping is found in the configuration file, the script automatically switches to the `Automatic` location. This requires the same sudoers configuration as above.

## Configuration UI

A command-line configuration tool is available for managing LocationChanger.

<img width="1066" height="862" alt="Bash Configuration UI" src="https://github.com/user-attachments/assets/9d32b9bb-62c0-4978-8b90-605222bc993c" />

### Use the configuration UI

```bash
./config-ui.sh
```

This interactive tool provides the following:

#### Menu options

1. **Add SSID mapping**: Create new Wi-Fi to location mappings
2. **Remove SSID mapping**: Delete existing mappings
3. **Refresh current SSID**: Update the current Wi-Fi status
4. **View configuration file**: Display the current configuration
5. **Install LocationChanger**: Run the installation script and preserve existing mappings
6. **Uninstall LocationChanger**: Completely remove all files and configurations
7. **Exit**: Quit the configuration tool

#### Features

- A colorized terminal interface with clear status displays
- Current Wi-Fi status and mapping count
- Installation that preserves existing mappings
- Complete uninstall support
- Error handling with direct feedback
- Permission-aware file operations

#### Example usage

```bash
# Start the configuration tool
./config-ui.sh

# The tool shows:
# - Current Wi-Fi status
# - Existing mappings
# - An interactive menu for all operations
```

#### Install through the UI

1. Run `./config-ui.sh`
2. Select option 5, **Install LocationChanger**
3. The tool will:
   - Check for existing mappings
   - Run the installation script
   - Restore your configurations
   - Provide the next sudoers step

#### Uninstall through the UI

1. Run `./config-ui.sh`
2. Select option 6, **Uninstall LocationChanger**
3. Confirm the removal
4. The tool will:
   - Stop the service
   - Remove all files
   - Clean up configurations
   - Remind you to clean up sudoers manually

## Configure manually instead

If you prefer manual configuration, create a configuration file from the sample:

```bash
cp ./locationchanger.conf.sample ./locationchanger.conf
```

Add one line for each location and SSID pair that this service should recognize and activate. Each line contains a location name and a Wi-Fi SSID separated by a space. Use exact capitalization, and use quotes when needed.

Example:

```text
home myWifiName
```

If your SSID contains spaces, wrap it in quotes:

```text
home "Wu Tang LAN"
```

Use the exact location names as they appear in **System Settings** → **Network**, and use the exact SSID names shown in the Wi-Fi menu. Capitalization and spacing must match.

## Required setup

Manual setup is still required for location switching:

1. Run `./install.sh` to install the helper script
2. Add this line to your sudoers file:

   ```bash
   sudo visudo
   ```

   Replace `your_username` with your actual username:

   ```text
   your_username ALL=(ALL) NOPASSWD: /usr/local/bin/locationchanger-helper
   ```

   To find your username, run `whoami`.
