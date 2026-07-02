# Windows 11 & Kali Linux Administration Lab Notes

A comprehensive technical guide summarizing user management, security architectures, permissions, and registry manipulation across Windows 11 Home and Kali Linux.

---

## 🛠️ Part 1: Windows 11 Systems Administration

### 1. Operating System Edition Limitations
Windows 11 Home lacks native access to advanced administrative management consoles found in Professional or Enterprise editions.
* **Missing Consoles:** Local Security Policy (`secpol.msc`) and Group Policy Editor (`gpedit.msc`).
* **The Solution:** These management packages can be injected manually using Deployment Image Servicing and Management (DISM) via an elevated Command Prompt:
  ```cmd
  FOR %F IN ("%SystemRoot%\servicing\Packages\Microsoft-Windows-GroupPolicy-ClientTools-Package~*.mum") DO (DISM /Online /NoRestart /Add-Package:"%F")
  FOR %F IN ("%SystemRoot%\servicing\Packages\Microsoft-Windows-GroupPolicy-ClientExtensions-Package~*.mum") DO (DISM /Online /NoRestart /Add-Package:"%F")
  ```

### 2. Windows Licensing & Identity Persistence
* **OEM Activation:** Factory-built computers embed the Windows activation license key directly into the motherboard's firmware (ACPI table / UEFI). If the operating system breaks or the hard drive is replaced, Windows reads the hardware identifier upon reinstallation and activates automatically over the internet.
* **Account Provisioning:** Microsoft Accounts handle authorization clouds differently than local hardware tags. If an account holds a digital entitlement for a Windows Pro upgrade, signing into that specific profile can trigger an OS elevation.

### 3. Registry Manipulation (Cosmetic Modification)
System registration metadata is stored deterministically in the Windows Registry. Modifying these specific keys changes OS information layout (e.g., inside the `winver` dialog) but does not alter functional profile settings or license integrity.

* **Registry Path:** `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion`
* **Key Components:**
  * `RegisteredOwner` (String Value): Defines the primary operating identity.
  * `RegisteredOrganization` (String Value): Defines organizational infrastructure (optional; can be manually created via `New -> String Value`).

---

## 🐧 Part 2: Kali Linux User & Group Architecture

### 1. Account Creation and Privilege Escalation
User account creation requires exact command syntax and proper token separation.

* **Command Prompt (Windows) User Provisioning Syntax:**
  ```cmd
  net user <username> <password> /add
  ```
  *Note: A strict whitespace delimiter must precede the `/add` switch, otherwise the compiler parses the flag as part of the string payload.*

* **Linux User Provisioning:**
  Using `adduser <username>` invokes an interactive script that creates the user profile, home directory, and requests optional finger data fields (Full Name, Room Number, etc.) which can be bypassed safely via `ENTER`.

* **Privilege Elevation via `sudo` (SuperUser DO):**
  The `sudo` command permits authorized users to execute binaries with security privileges of another user (normally `root`). To authorize an account, it must be appended to the administrative `sudo` group:
  ```bash
  sudo usermod -aG sudo <username>
  ```

### 2. Subverting User Shell Environments
To dynamically switch active shell profiles within a unified terminal session, use the Substitute User (`su`) command.
```bash
su - <username>
```
* **The Hyphen (`-`) Operator:** Explicitly instructs the shell to initialize a complete login environment, resetting environment variables, execution paths (`$PATH`), and changing the working directory to the target user's home directory (`~/`).

### 3. Group Security Architecture & Policy
Linux enforces data and security boundaries utilizing the **Principle of Least Privilege**.

#### Default Provisioning
By default, a newly provisioned user profile is completely isolated. It is only placed into a newly generated **Primary Group** bearing its own name (`username:username`) and possesses zero system execution capabilities.

#### Checking System Groups
Group identities are plain-text string definitions saved locally on the hard disk.
* **Config File Location:** `/etc/group`
* **Terminal Diagnostics:**
  ```bash
  # Check groups for the current session user
  groups
  
  # Check groups for a specific account
  groups <username>
  
  # Output a raw list of all unique group strings on the OS
  cut -d: -f1 /etc/group
  ```

#### The Hazards of Global Group Assignment
Adding a human user profile to all native system groups collapses the security infrastructure of the Unix architecture.
* **System Isolation Failure:** System daemons run under unique background groups (e.g., `www-data`, `systemd-journal`). Exposing a user to all groups allows compromise vectors to span across isolated processes.
* **The `disk` Group Danger:** Granting access to the `disk` system group allows a user to read or write raw binary segments directly to the storage controller blocks, entirely bypassing the Linux file system security layer and standard `sudo` authorization checks.

#### Best Practice: Sequential Batch Assignment
To securely add a user account to multiple required application groups simultaneously, pass a comma-separated array string via the append-group flag:
```bash
sudo usermod -aG audio,video,wireshark,docker <username>
```
* **Crucial Warning:** The `-a` (append) switch must always accompany `-G`. Failing to include `-a` instructs the system to completely overwrite the user's group array, instantly revoking all prior memberships (including `sudo`).
* **Session Flush:** Changes do not apply to currently loaded processes. The modified user must completely terminate their active session (log out) and re-authenticate for permissions to inject into the user token.
