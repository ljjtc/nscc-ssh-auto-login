# Windows SSH Auto-Login to NSCC Using DPAPI

Securely automate SSH password entry for NSCC on Windows using **Windows DPAPI** and **SSH_ASKPASS**, without repeatedly typing your password.

## Background

NSCC has disabled SSH key-based authentication, requiring users to enter their account password when connecting via SSH.

This can be inconvenient when frequently accessing NSCC through PowerShell, Command Prompt, or VS Code Remote-SSH, especially when reconnecting after a dropped connection.

This guide provides a workaround using:

- **Windows DPAPI** to securely store the NSCC password.
- **PowerShell** to retrieve the encrypted credential.
- **SSH_ASKPASS** to automatically provide the password to Windows OpenSSH.
- **VS Code Remote-SSH** integration for password-free interaction after initial setup.

The password is encrypted using Windows DPAPI and is not stored as plain text. After the initial configuration, you normally only need to log in to Windows to use the saved credential.

**Important:** This method does not bypass NSCC authentication. It automatically supplies your existing NSCC password when requested. Check that this form of credential storage and automated authentication is permitted by your institution.

## Prerequisites

- Windows 10 or Windows 11
- Windows PowerShell
- Windows OpenSSH Client
- A valid NSCC account
- A working SSH configuration with the host alias `nscc`

You should already be able to connect using:

```powershell
ssh nscc
```

If you use a jump host or a custom SSH configuration, keep that configuration unchanged.

Throughout this guide, replace `YOUR_NSCC_USERNAME` with your actual NSCC username.

## 1. Save the NSCC Password Securely

Open **PowerShell** and create the SSH configuration directory:

```powershell
New-Item -ItemType Directory -Force "$HOME\.ssh" | Out-Null
```

Create an encrypted credential file:

```powershell
$pw = Read-Host "NSCC SSH password" -AsSecureString
$cred = [System.Management.Automation.PSCredential]::new("YOUR_NSCC_USERNAME", $pw)
$cred | Export-Clixml "$HOME\.ssh\nscc-cred.xml"
```

Replace `YOUR_NSCC_USERNAME` with your actual NSCC username.

When prompted:

```text
NSCC SSH password:
```

Enter your normal NSCC password and press Enter.

Nothing will appear on the screen while typing. This is normal.

Verify that the credential file exists:

```powershell
Test-Path "$HOME\.ssh\nscc-cred.xml"
```

Expected output:

```text
True
```

The encrypted credential is stored at:

```text
C:\Users\<WindowsUsername>\.ssh\nscc-cred.xml
```

**Security:** The password is protected by Windows DPAPI and can normally be decrypted only under the same Windows user account on the same computer.

## 2. Create the SSH Askpass PowerShell Script

Run the following command in PowerShell:

```powershell
@'
param([string]$Prompt)

if ($Prompt -notmatch '(?i)YOUR_NSCC_USERNAME@.*nscc.*password') {
    exit 1
}

$cred = Import-Clixml "$env:USERPROFILE\.ssh\nscc-cred.xml"
$cred.GetNetworkCredential().Password
'@ | Set-Content "$HOME\.ssh\ssh-askpass.ps1" -Encoding UTF8
```

Replace `YOUR_NSCC_USERNAME` with your NSCC username.

The script:

1. Receives the password prompt from SSH.
2. Checks whether the prompt appears to correspond to your NSCC account.
3. Loads the encrypted credential from `nscc-cred.xml`.
4. Decrypts the password using Windows DPAPI.
5. Returns the password to SSH.

The prompt check reduces accidental use of the stored password for unrelated SSH connections.

**Note:** The prompt check is not a strong security boundary. A malicious server could imitate an NSCC password prompt. Always verify SSH host keys and protect your SSH configuration.

## 3. Create the Windows Askpass Wrapper

Windows OpenSSH can invoke a wrapper script to retrieve the password.

Create the wrapper:

```powershell
@'
@echo off
powershell.exe -NoProfile -ExecutionPolicy Bypass -File "%USERPROFILE%\.ssh\ssh-askpass.ps1" "%~1"
'@ | Set-Content "$HOME\.ssh\ssh-askpass.cmd" -Encoding ASCII
```

You should now have three files:

```text
C:\Users\<WindowsUsername>\.ssh\
    nscc-cred.xml
    ssh-askpass.ps1
    ssh-askpass.cmd
```

| File | Purpose |
|---|---|
| `nscc-cred.xml` | Stores the DPAPI-encrypted credential |
| `ssh-askpass.ps1` | Retrieves the password after checking the SSH prompt |
| `ssh-askpass.cmd` | Allows Windows OpenSSH to invoke the PowerShell script |

## 4. Test the Configuration Temporarily

Before enabling automatic password entry permanently, test it in the current PowerShell session.

If this is your first connection, verify and accept NSCC's SSH host key through a normal manual login before enabling automatic password entry.

Set the temporary environment variables:

```powershell
$env:SSH_ASKPASS="$HOME\.ssh\ssh-askpass.cmd"
$env:SSH_ASKPASS_REQUIRE="force"
```

Verify:

```powershell
$env:SSH_ASKPASS
$env:SSH_ASKPASS_REQUIRE
```

Expected output:

```text
C:\Users\<WindowsUsername>\.ssh\ssh-askpass.cmd
force
```

Now connect:

```powershell
ssh nscc
```

If everything is configured correctly, Windows OpenSSH should automatically retrieve the stored password and log in to NSCC.

If SSH asks for the password manually or authentication fails, verify the username, password, SSH prompt format, and OpenSSH configuration.

The askpass prompt-matching expression may need adjustment if your SSH configuration produces a different password prompt.

## 5. Make the Configuration Permanent

After confirming that the temporary test works, save the environment variables for your Windows user account.

Run in PowerShell:

```powershell
[Environment]::SetEnvironmentVariable(
    "SSH_ASKPASS",
    "$HOME\.ssh\ssh-askpass.cmd",
    "User"
)

[Environment]::SetEnvironmentVariable(
    "SSH_ASKPASS_REQUIRE",
    "force",
    "User"
)
```

You only need to do this once.

Completely close and reopen PowerShell, Command Prompt, and VS Code so they inherit the updated environment variables.

### Verify in PowerShell

```powershell
$env:SSH_ASKPASS
$env:SSH_ASKPASS_REQUIRE
```

Expected output:

```text
C:\Users\<WindowsUsername>\.ssh\ssh-askpass.cmd
force
```

### Verify in Command Prompt

```bat
echo %SSH_ASKPASS%
echo %SSH_ASKPASS_REQUIRE%
```

The environment variables apply to both PowerShell and CMD.

**Note:** Because `SSH_ASKPASS_REQUIRE=force` is configured at the user level, it can affect other SSH connections as well. The helper intentionally refuses password prompts that do not match the configured NSCC pattern.

## 6. Connect to NSCC

Once configured, simply run:

```powershell
ssh nscc
```

The authentication process is:

```text
ssh nscc
    |
    v
Windows OpenSSH connects to NSCC
    |
    v
NSCC requests a password
    |
    v
SSH invokes ssh-askpass.cmd
    |
    v
ssh-askpass.ps1 checks the password prompt
    |
    v
nscc-cred.xml is decrypted using Windows DPAPI
    |
    v
The saved password is returned to SSH
    |
    v
Authentication succeeds
```

No additional GPG password or master password is required.

After restarting Windows, simply log in to your Windows account and run:

```powershell
ssh nscc
```

## 7. VS Code Remote-SSH Integration

If you use **VS Code Remote-SSH**, you can configure it to use the same Windows OpenSSH client and saved credential.

### Open VS Code settings

Press:

```text
Ctrl + Shift + P
```

Select:

```text
Preferences: Open User Settings (JSON)
```

Add the following settings:

```json
"remote.SSH.path": "C:\\Windows\\System32\\OpenSSH\\ssh.exe",
"remote.SSH.useLocalServer": true,
"remote.SSH.externalSSH_ASKPASS": true
```

If you use a specific SSH configuration file, also add:

```json
"remote.SSH.configFile": "C:\\Users\\<WindowsUsername>\\.ssh\\config"
```

Replace `<WindowsUsername>` with your Windows username.

### Example configuration

```json
{
    "remote.SSH.configFile": "C:\\Users\\<WindowsUsername>\\.ssh\\config",
    "remote.SSH.path": "C:\\Windows\\System32\\OpenSSH\\ssh.exe",
    "remote.SSH.useLocalServer": true,
    "remote.SSH.externalSSH_ASKPASS": true
}
```

If your `settings.json` already contains other settings, add these entries to the existing JSON object instead of replacing the entire file.

The important setting is:

```json
"remote.SSH.externalSSH_ASKPASS": true
```

This is intended to allow VS Code Remote-SSH to use the external SSH_ASKPASS configuration rather than replacing it with VS Code's own password prompt.

**Note:** Remote-SSH behavior and available settings may vary across extension versions. If automatic authentication does not work in VS Code, first confirm that `ssh nscc` works in a newly opened PowerShell session, then inspect the Remote-SSH output log.

### Connect from VS Code

Completely close and restart VS Code after updating the settings.

Press:

```text
Ctrl + Shift + P
```

Select:

```text
Remote-SSH: Connect to Host...
```

Choose:

```text
nscc
```

If your installed Remote-SSH version supports the configuration, VS Code should use the saved NSCC password automatically.

## 8. Security Considerations

### Password encryption

The NSCC password is not stored as plain text.

The credential file:

```text
nscc-cred.xml
```

uses Windows DPAPI encryption.

The encrypted credential is tied to the Windows user context and computer where it was created.

### Environment variables

The environment variables:

```text
SSH_ASKPASS
SSH_ASKPASS_REQUIRE
```

do not contain your password.

They only specify how OpenSSH should request passwords.

### Prompt validation

The script checks the SSH password prompt using:

```powershell
'(?i)YOUR_NSCC_USERNAME@.*nscc.*password'
```

This helps avoid accidentally submitting the password to an unrelated SSH connection.

However, this check does not verify server identity. SSH host-key validation remains essential.

### Windows account security

If another person gains access to your unlocked Windows account and can execute programs as your user, they may also be able to decrypt your stored credential.

For this reason:

- Keep your Windows account protected.
- Lock your computer when unattended.
- Do not share your encrypted credential file.
- Do not upload `nscc-cred.xml` to GitHub or any public repository.
- Do not disable SSH host-key verification.
- Delete and recreate the credential file whenever your NSCC password changes.

This method provides **convenience through encrypted credential storage**, not passwordless authentication in the cryptographic sense.

## 9. Disable Automatic Password Entry

To stop automatic password entry, remove the permanent environment variables.

Run in PowerShell:

```powershell
[Environment]::SetEnvironmentVariable(
    "SSH_ASKPASS",
    $null,
    "User"
)

[Environment]::SetEnvironmentVariable(
    "SSH_ASKPASS_REQUIRE",
    $null,
    "User"
)
```

For the current PowerShell session, also run:

```powershell
Remove-Item Env:SSH_ASKPASS -ErrorAction SilentlyContinue
Remove-Item Env:SSH_ASKPASS_REQUIRE -ErrorAction SilentlyContinue
```

Restart PowerShell, CMD, and VS Code afterward.

SSH will return to its normal password-prompt behavior.

## 10. Remove the Saved NSCC Credential

To remove the stored credential and helper scripts entirely:

```powershell
Remove-Item "$HOME\.ssh\nscc-cred.xml" -ErrorAction SilentlyContinue
Remove-Item "$HOME\.ssh\ssh-askpass.ps1" -ErrorAction SilentlyContinue
Remove-Item "$HOME\.ssh\ssh-askpass.cmd" -ErrorAction SilentlyContinue
```

You should also remove the environment variables as described in Section 9.

Afterward, connecting with:

```powershell
ssh nscc
```

will require manual password entry again.

## Final Usage

After successful configuration, the normal workflow is simply:

**PowerShell or Command Prompt:**

```powershell
ssh nscc
```

**VS Code:**

```text
Remote-SSH: Connect to Host... → nscc
```

Your NSCC password can be supplied automatically using Windows DPAPI and SSH_ASKPASS, reducing repetitive password entry while preserving encrypted storage of the credential.

---

*This guide is an independent user workaround and is not an official NSCC authentication procedure. Users remain responsible for complying with applicable NSCC and institutional security policies.*
