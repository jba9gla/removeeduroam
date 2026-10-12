# Remove eduroam

A small macOS tool that removes the **eduroam** Wi-Fi network and its saved keychain entries, so you can reconnect and sign in again from scratch. It is useful when eduroam has stopped connecting or keeps using an old password.

It comes in two forms that share one script:

- **A double-click app** (`Remove eduroam.app`) with plain dialogs. It always runs a check first and only offers to remove eduroam if every check passes.
- **A Terminal command** for people who prefer it, or for IT staff.

Download page for end users: https://jacb.co.uk/removeeduroam/

> **Use at your own risk.** We have taken care to make sure it works, but it is provided as is, with no warranty of any kind. To the fullest extent permitted by law, JACB Services accepts no liability for any loss or problems that result from using it. The app and the script both ask the user to accept this before anything is removed.

## Requirements

- macOS, with an administrator account (the tool needs admin rights to change the System keychain)
- A personal Mac. It refuses to run on Macs enrolled in an organisation's MDM unless `--allow-managed` is given

## Using the app

1. Open **Remove eduroam**.
2. Read the message and click **I Accept**. The app runs a check, which changes nothing.
3. Enter your Mac password when asked.
4. If every line of the results says PASS, click **Remove eduroam**.
5. Reconnect to eduroam from the Wi-Fi menu and sign in again.

If eduroam is not saved on the Mac, the check says so and the app stops. If the Mac is managed by an organisation, the app stops and tells the user to ask their IT team.

## Using Terminal

Check first. This changes nothing:

```sh
curl -fsSL https://jacb.co.uk/removeeduroam/removeeduroam.sh -o "$TMPDIR/removeeduroam.sh" && \
  sudo bash "$TMPDIR/removeeduroam.sh" --test; rm -f "$TMPDIR/removeeduroam.sh"
```

If every line of the results says PASS, remove eduroam:

```sh
curl -fsSL https://jacb.co.uk/removeeduroam/removeeduroam.sh -o "$TMPDIR/removeeduroam.sh" && \
  sudo bash "$TMPDIR/removeeduroam.sh" --real; rm -f "$TMPDIR/removeeduroam.sh"
```

The script downloads to a file first, rather than piping `curl` into `bash`, so a dropped connection can't run half a script. `$TMPDIR` is a per-user folder on macOS, so other users can't swap the file in a shared `/tmp`.

A plain `--real` run shows the terms in a dialog and needs the user to click **I Accept**. `--test` changes nothing, so it doesn't ask.

## Options

| Option | What it does |
| --- | --- |
| `--test` (or `--dry-run`) | Runs the checks and shows what would be removed. Changes nothing. |
| `--real` | Removes the network and its keychain entries. |
| `--ssid NAME` | Use a different network name. The default is `eduroam`. Handy for testing against a fake network. |
| `--accept` | Accepts the terms without the dialog. Only use it if you have read them. The app passes this after showing its own terms dialog. |
| `--allow-managed` | Runs even on a Mac managed by an organisation. For IT staff. |
| `--app` | Used by the app. Sends progress to the log only and prints just the summary. |
| `--version` | Prints the version. |
| `-h`, `--help` | Shows the usage. |

You must give either `--test` or `--real`.

### Exit codes

| Code | Meaning |
| --- | --- |
| 0 | Success (the check passed, or the removal finished) |
| 1 | An error, a failed check, no Wi-Fi interface, or not run as root |
| 2 | Bad options |
| 3 | The terms were not accepted |
| 4 | The Mac is managed by an organisation |
| 10 | Nothing to remove (only returned in `--app` mode) |
| 130 | Interrupted |

## What it does

- Removes the network from the Mac's list of known Wi-Fi networks, using `networksetup`.
- Deletes keychain items labelled with the network name, `com.apple.network.eap.user.item.wlan.ssid.<name>` or `com.apple.network.eap.system.item.wlan.ssid.<name>`. It looks for both generic and internet passwords, in the System keychain and the logged-in user's login keychain.
- Warns if a configuration profile mentioning the network is still installed. It does **not** remove profiles. They are removed in System Settings, Privacy & Security, Profiles.
- Never touches any other Wi-Fi network.
- Collects and sends nothing. It runs locally and the only network request is the app or the Terminal command fetching the script.

Deleted keychain entries can't be restored. Users need their eduroam sign-in details to connect again afterwards.

## Log file

The log is saved to `~/Library/Logs/removeeduroam.log`, or `/var/log/removeeduroam.log` if that can't be written. It records what was checked and removed, the macOS version, the Mac's username and computer name, and that the terms were accepted. It never contains passwords. Users should look it over before sending it to anyone.

## Security notes

The tool runs as root, so the way the script is delivered matters.

- **The app pins the script.** The app contains the SHA-256 fingerprint of `removeeduroam.sh` and refuses to run a downloaded script that doesn't match. If the server were ever compromised, the app would not run the changed file. `build-app.sh` fills the fingerprint in automatically, so any change to the script means rebuilding, signing and notarizing the app, and uploading the script **unchanged and in binary mode**.
- **The Terminal route can't be pinned.** Anyone using it is trusting the server. Protect the hosting account with two-factor authentication and unique passwords, and use SFTP only.
- **Download first, then run.** See above.
- **Managed Macs.** Deleting system-wide Wi-Fi credentials could break a Mac managed by an organisation, so the tool refuses by default.
- **Only use the official address.** Anyone can copy a page like this. Link to it from a source your users trust.

## Building the app

Building needs a Mac with the Xcode command line tools. Signing and notarizing need an Apple Developer ID.

Files needed in one folder: `RemoveEduroam.applescript`, `removeeduroam.sh`, `applet.icns` and `build-app.sh`.

For testing, build without signing:

```sh
./build-app.sh --unsigned
```

For a release:

1. Put your certificate name in `SIGN_ID` at the top of `build-app.sh`, for example `Developer ID Application: Your Name (TEAMID)`.
2. Store notary credentials once:
   ```sh
   xcrun notarytool store-credentials "notary-profile"
   ```
3. Run:
   ```sh
   ./build-app.sh
   ```

This compiles the AppleScript, fills in the script fingerprint and version, adds the icon, signs with the hardened runtime, notarizes, staples and produces `Remove-eduroam.zip`. Upload that zip, and upload `removeeduroam.sh` exactly as it is in the build folder.

The version number lives in one place, the `VERSION` line at the top of `removeeduroam.sh`.

## Repository layout

```
removeeduroam.sh              The script that does the work
RemoveEduroam.applescript     Source of the app (dialogs and the download step)
build-app.sh                  Builds, signs and notarizes the app
applet.icns                   App icon
web/index.html                The download page
web/icon.png                  Icon used on the page
```

## Testing

The script can be exercised without touching real network settings. Create a fake network and keychain entry, then run the check and the removal with `--ssid`:

```sh
sudo networksetup -addpreferredwirelessnetworkatindex en0 "eduroam-test" 0 WPA2
security add-generic-password -a test -s "eduroam-test" -l "eduroam-test" -w dummy ~/Library/Keychains/login.keychain-db

sudo bash removeeduroam.sh --test --ssid eduroam-test
sudo bash removeeduroam.sh --real --ssid eduroam-test
```

## Licence

See the `LICENSE` file.
