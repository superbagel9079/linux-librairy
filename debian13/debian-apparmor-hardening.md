# Debian 13 AppArmor enforcement

AppArmor confines programs to the files, capabilities and namespaces their
profiles allow. On the Debian 13 gateway VM it ships enabled, with profiles
loaded in several modes; this note records that state and how the complain
profiles were moved to enforce. Established 2026-10-10 against apparmor 4.1.0-1.

## Chapter I. State before

### Part I. Modes

`sudo aa-status` reported 105 profiles loaded: 6 in enforce, 20 in complain and
76 in unconfined mode. The 76 unconfined are shipped that way by Debian, with
`flags=(unconfined)`, mostly desktop and browser applications.

### Part II. The tools are a separate package

`aa-status` lives in the `apparmor` package, but `aa-enforce` and `aa-complain`
live in `apparmor-utils`, which is not installed by default. A bare
`sudo aa-enforce` therefore fails with `command not found`: it is a missing
package, not a PATH problem.

## Chapter II. Enforcement

```
sudo apt install apparmor-utils
sudo aa-enforce $(grep -rl 'flags=.*complain' /etc/apparmor.d/)
```

The targeted form moves only the profiles the package ships in complain and
leaves the 76 unconfined profiles alone, since enforcing those risks breaking
installed desktop or browser applications. 19 profiles were moved, among them
`unix-chkpwd`, `unprivileged_userns`, `Xorg` and `plasmashell`.

## Chapter III. Verification and rollback

```
sudo aa-status
sudo ausearch -m AVC
sudo journalctl -k | grep -i apparmor
```

Denials appear as AVC records in auditd, which is enabled on this VM. To release
one program, `sudo aa-complain /etc/apparmor.d/<profile>`.

## Chapter IV. Caveats

- `Xorg` and `plasmashell` were enforced. On a headless gateway they are inert;
  on a machine with a graphical session they can break it, and
  `sudo aa-complain /etc/apparmor.d/Xorg` reverts them.
- `unprivileged_userns` is the profile worth enforcing on a server: it audits
  and denies `capability` and `change_profile` inside new unprivileged user
  namespaces, a common path for kernel exploits.

## Sources

- [apparmor.d(5) - Debian trixie](https://manpages.debian.org/trixie/apparmor/apparmor.d.5.en.html): profile flags, including complain and unconfined.
- Evidence: `sudo aa-status` and the profile files under `/etc/apparmor.d` on the gateway VM, 2026-10-10.
