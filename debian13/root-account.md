# Root account hardening

Root is removed from every password-based login path, local and remote. Root
stays reachable only through sudo from an account in the sudo group.

## What is in place

- `/etc/ssh/sshd_config.d/00-hardening.conf` sets `PermitRootLogin no`, so there
  is no SSH login as root, neither by password nor by key.
- `passwd -l root` disables the root password: it prepends `!` to the hash in
  `/etc/shadow`, which blocks the console login, `su`, and any PAM password
  check for root.
- Elevation goes only through sudo. `/etc/sudoers` is the untouched package
  default and grants the sudo group; `/etc/sudoers.d/00-hardening` adds the
  logging policy.

## Why

A password on root is the highest-value credential on the host and the most
attacked one over SSH. Removing it leaves no password to guess, and every
privileged action is then attributable to a named admin account through sudo.

## Verification

```
sudo passwd -S root                              # expect: root L ...
sudo getent shadow root | cut -d: -f2 | head -c1 # expect: !
sudo -l -U <admin>
ssh root@<host>                                  # expect: Permission denied
```

`L` in the `passwd -S` output means locked. The `!` at the start of the shadow
field is the lock marker.

## Reverting

```
sudo passwd -u root    # removes the ! and restores any previous password
sudo passwd root       # or set a new password
```

## Caveats

- Locking root does not affect sudo. It only removes root's own password login.
- Keep at least one account in the sudo group, and an out-of-band console
  (serial or hypervisor) reachable, before locking. If the admin account loses
  both its password and its sudo right, that console is the only way back in.
- Physical or hypervisor console access still yields root regardless of this
  setting. This note is about network and password paths, not about physical
  access.
- On a fresh Debian install root has no password set until someone runs
  `passwd root`. Running `passwd -l root` early makes the intent explicit and
  keeps it locked across reinstalls of the SSH config.
