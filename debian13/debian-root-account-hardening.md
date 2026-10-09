---
date: 2026-10-10
status: draft
subject: Debian 13 root account hardening on the gateway VM
tags: [debian, hardening]
link: ""
---

# Debian 13 root account hardening

Root is taken out of every password-based login path, local and remote, on the
Debian 13 gateway VM; root stays reachable only through sudo from a named admin
account. Established 2026-10-10 against Debian 13 (trixie), OpenSSH 10.0p2.

## Chapter I. What is in place

### Part I. Remote login

`PermitRootLogin no` in `/etc/ssh/sshd_config.d/00-hardening.conf` removes root
from SSH, by password and by key. Verified with `sudo sshd -T`, which returns
`permitrootlogin no`.

### Part II. Local password

`passwd -l root` disables root's password: the manpage states it "adds a ! at
the beginning of the password", a value matching no possible encrypted value.
Verified with `sudo passwd -S root`, which returns `root L`, the L meaning
locked.

### Part III. Elevation

Elevation is only through sudo. `/etc/sudoers` is the untouched package default
granting the sudo group, and `/etc/sudoers.d/00-hardening` adds the logging
policy, so every privileged action is attributable to a named account.

## Chapter II. Verification

```
sudo passwd -S root                              # expect: root L ...
sudo getent shadow root | cut -d: -f2 | head -c1 # expect: !
sudo -l -U <admin>
ssh root@<host>                                  # expect: Permission denied
```

The `!` at the start of the shadow field is the lock marker.

## Chapter III. Reverting

```
sudo passwd -u root    # removes the ! and restores any previous password
sudo passwd root       # or set a new password
```

## Chapter IV. Caveats

- Locking root does not affect sudo, it only removes root's own password login.
- Keep at least one account in the sudo group, and an out-of-band console
  (serial or hypervisor) reachable, before locking. If the admin account loses
  both its password and its sudo right, that console is the only way back in.
- Physical or hypervisor console access still yields root regardless of this
  setting. This note is about network and password paths, not physical access.
- A fresh Debian install has no root password until someone runs `passwd root`.
  Running `passwd -l root` early makes the intent explicit and keeps it locked
  across reinstalls of the SSH config.

## Sources

- [passwd(1) - Debian trixie](https://manpages.debian.org/trixie/passwd/passwd.1.en.html): the `-l` option adds a `!` at the beginning of the password and does not disable the account for other tokens such as an SSH key.
- [sshd_config(5) - Debian trixie](https://manpages.debian.org/trixie/openssh-server/sshd_config.5.en.html): `PermitRootLogin no` refuses login as root over SSH.
- Evidence: `sudo passwd -S root` and `sudo sshd -T` on the gateway VM, 2026-10-10.

## Related

- [[debian-apparmor-hardening]]: the other access-control note for the same VM.
