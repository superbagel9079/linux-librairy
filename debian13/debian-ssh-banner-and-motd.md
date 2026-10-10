# Debian 13 SSH banner and login message

Two separate texts greet an SSH client on the Debian 13 gateway VM: a banner
shown before authentication, which every client sees, and a message shown after
a successful login. This note records where each comes from and how they were
replaced. Established 2026-10-10 against OpenSSH 10.0p2.

## Chapter I. Pre-authentication banner

### Part I. Source

sshd sends the contents of the file named by the `Banner` keyword before
authentication is allowed. The file is emitted verbatim, so it must hold the
text alone, with no comment character and no tabulation, and read as ASCII.

### Part II. Setting

`Banner /etc/ssh/banner` in `/etc/ssh/sshd_config.d/00-hardening.conf` points
sshd at the warning text. `ssh -T` style checks confirm it with
`sshd -T | grep banner`.

## Chapter II. Post-login message

### Part I. Three sources

- The kernel line, `Linux <host> <release> ...`, comes from
  `/etc/update-motd.d/10-uname`.
- The Debian legal block, `The programs included with the Debian ...`, comes
  from `/etc/motd`, the base-files default.
- The `Last login: ...` line comes from sshd, option `PrintLastLog`, yes by
  default.

### Part II. Mechanism

Debian's `pam_motd` runs `run-parts --lsbsysinit /etc/update-motd.d`, writes the
result to `/run/motd.dynamic`, prints that file, then prints `/etc/motd`.
Verified by the strings carried by `pam_motd.so`, which name both
`/etc/update-motd.d` and `/run/motd.dynamic`.

## Chapter III. Removing the dynamic lines

### Part I. chmod on the script is not reliable

`chmod -x /etc/update-motd.d/10-uname` failed with `new permissions are
rw-r--r-x, not rw-r--r--`, so the execute bit stayed. `lsattr` showed only the
`e` extent flag, no immutability, and `getfacl` was unavailable because the
`acl` package is not installed. The cause was never established.

### Part II. Rename instead, verified against run-parts

`run-parts` ignores any filename containing a dot. Renaming the script is
therefore enough to disable it, without touching its mode. Verified: with both
`10-uname` and `10-uname.off` present and executable, `run-parts --lsbsysinit`
ran only the first.

```
sudo mv /etc/update-motd.d/10-uname /etc/update-motd.d/10-uname.off
sudo mv /etc/update-motd.d/92-unattended-upgrades /etc/update-motd.d/92-unattended-upgrades.off
sudo rm -f /run/motd.dynamic /run/motd.dynamic.new
```

The `rm` matters: `chmod` or `mv` changes the ctime, not the mtime, and
`pam_motd` rebuilds the cache only when a script is newer than it or the cache
is missing. Removing the cache forces a rebuild at the next login.

### Part III. The replacement message

`/etc/motd` is replaced with the access warning, and `PrintLastLog no` in the
sshd drop-in removes the last-login line. `DebianBanner` is unrelated: the
manpage states it only sets the distribution version suffix in the initial
handshake, not any banner text.

## Chapter IV. Verification

```
sudo sshd -T | grep -E '^(banner|printlastlog|printmotd)'
cat /etc/motd
ls -l /etc/update-motd.d/
```

On reconnect the client sees the pre-authentication banner, then, after login,
only the content of `/etc/motd`: no kernel line, no Debian legal block, no
`Last login` line.

## Sources

- [sshd_config(5) - Debian trixie](https://manpages.debian.org/trixie/openssh-server/sshd_config.5.en.html): `Banner` sends the named file before authentication, `PrintLastLog` prints the last-login line, `DebianBanner` only affects the handshake version suffix.
- [passwd(1) - Debian trixie](https://manpages.debian.org/trixie/passwd/passwd.1.en.html): consulted for the account lock command on the sibling note.
- Evidence: `strings /usr/lib/x86_64-linux-gnu/security/pam_motd.so`, a `run-parts --lsbsysinit` test, and the `chmod` failure followed by the rename, on the gateway VM, 2026-10-10.
