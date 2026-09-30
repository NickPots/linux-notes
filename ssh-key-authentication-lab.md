# SSH Key Authentication Lab: Hardening a Small Home Network

Notes from setting up passwordless, key-only SSH between three machines on a home LAN, and locking down the SSH daemon on each one.

## Goal

- Log in between all machines over SSH **without a password**
- Disable password authentication on every machine
- Restrict the firewall so SSH is only reachable from the local network

## Lab environment

The lab consists of three machines: one desktop PC running **Arch Linux**, a second PC running **Ubuntu**, and one **MacBook** running macOS. Each machine acts as both an SSH server and an SSH client, so every direction of login had to be set up and tested separately.

| Machine | OS | Role |
|---------|----|------|
| Desktop PC | Arch Linux | SSH server and client |
| Second PC | Ubuntu | SSH server and client |
| Laptop | macOS | SSH server and client |

All machines are on the same private subnet (`192.168.x.0/24`) behind a home router. No port forwarding is configured, so nothing is exposed to the internet.

## Key concept: lock and key

The part that confused me at first is which key goes where. A mental model that helped:

- The **public key** is the **lock**. It can be copied anywhere.
- The **private key** is the **key**. It never leaves the machine you connect *from*.

To log in **from A to B**:

1. A holds the private key.
2. B holds the public key in `~/.ssh/authorized_keys`.

So the public key is always copied to the machine you want to log **in to**. The password is only used once, to prove ownership of the account while installing the lock.

There are two separate trust relationships:

- **Server trusts client:** the client's public key is in `authorized_keys` on the server.
- **Client trusts server:** the server's fingerprint is accepted on first connect and stored in `~/.ssh/known_hosts`.

## Step 1: Check the network and SSH daemon

```bash
ip -br a                       # find the machine's IP address
systemctl status sshd          # Arch (service is called "ssh" on Ubuntu)
ss -tlnp | grep :22            # confirm sshd is listening on port 22
```

Install and enable OpenSSH if needed:

```bash
sudo pacman -S openssh         # Arch
sudo systemctl enable --now sshd
```

On macOS, enable **Remote Login** under System Settings, General, Sharing.

## Step 2: Generate a key pair

On each machine that will connect to others:

```bash
ssh-keygen -t ed25519
```

Use a passphrase. The private key is now the only thing protecting access, so it should be protected too.

## Step 3: Install the public key on the target

From the client machine:

```bash
ssh-copy-id user@<target-ip>
# or, for a key with a custom name:
ssh-copy-id -i ~/.ssh/<keyname>.pub user@<target-ip>
```

This logs in once with the password and appends the public key to `~/.ssh/authorized_keys` on the target. It never copies the private key.

Verify on the target:

```bash
cat ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

sshd refuses keys if these permissions are too open.

## Step 4: Firewall (ufw on Linux)

My Arch machine had ufw active with `Default: deny (incoming)` and no rules, which silently blocked all SSH. Symptom from the client: `Connection timed out`.

Rather than opening port 22 to everyone, allow only the local subnet:

```bash
sudo ufw allow from 192.168.x.0/24 to any port 22 proto tcp
sudo ufw status numbered
```

Useful diagnostics when a connection fails:

```bash
sudo ufw status verbose
sudo journalctl -k | grep "UFW BLOCK"     # blocked packets (needs logging on)
nc -zv <target-ip> 22                     # from the client
```

- `Connection refused`: nothing is listening, or there is a reject rule
- `Timed out`: packets are being dropped, typically by a firewall

## Step 5: Disable password authentication

**Before doing this, confirm key login works from every client.** Otherwise you can lock yourself out.

```bash
ssh -o PasswordAuthentication=no user@<target-ip>
```

If that logs in, the key is working and the password was not used as a fallback.

Then, on the target, create a drop-in config file:

```bash
ls /etc/ssh/sshd_config.d/                        # check for existing drop-ins
grep -i '^Include' /etc/ssh/sshd_config           # make sure drop-ins are loaded
sudo nano /etc/ssh/sshd_config.d/00-keyonly.conf
```

```
PasswordAuthentication no
KbdInteractiveAuthentication no
```

Notes:

- In sshd, the **first value wins**, and drop-ins are read alphabetically. Naming the file `00-...` makes sure it beats files like `50-cloud-init.conf` (which sets `PasswordAuthentication yes` on Ubuntu) or `99-archlinux.conf`.
- `KbdInteractiveAuthentication no` closes a back door where PAM could still accept passwords.
- A drop-in survives OS updates that overwrite the main `sshd_config`.

Validate and apply:

```bash
sudo sshd -t                                                              # syntax check
sudo sshd -T | grep -iE 'passwordauthentication|kbdinteractiveauthentication'   # both must say "no"
sudo systemctl restart sshd                                               # "ssh" on Ubuntu
```

On macOS there is no restart needed in most cases, since each new connection starts a fresh sshd process that re-reads the config.

**Keep your current session open** and test from a new terminal, so you can roll back if something goes wrong.

## Step 6: Verify that passwords are really disabled

Force ssh to skip key authentication so it has to fall back to a password:

```bash
ssh -o PubkeyAuthentication=no user@<target-ip>
```

Expected result:

```
user@<target-ip>: Permission denied (publickey).
```

There must be **no password prompt**. If `user@host's password:` appears, password login is still enabled somewhere.

Find the culprit with:

```bash
sudo grep -ri PasswordAuthentication /etc/ssh/
```

Then confirm that normal key login still works from every client.

## Step 7: SSH client config for convenience

A `~/.ssh/config` on the **client** lets you type `ssh arch-pc` instead of the full command. This file only applies to the machine you connect *from*.

```
Host arch-pc
    HostName 192.168.x.x
    User <username>
    IdentityFile ~/.ssh/<private-key-name>

Host ubuntu-pc
    HostName 192.168.x.x
    User <username>
```

```bash
chmod 600 ~/.ssh/config
```

`IdentityFile` is only needed when the private key is not named `id_ed25519` (or another default name).

## Mistakes and lessons learned

- **Pasted multiple commands at once.** When the first `ssh` command succeeded, the next line was typed into the remote session instead of running locally. Run one `ssh` test at a time and check which machine's prompt you are on.
- **Tested from the wrong machine.** Always run key-login tests from the machine you actually care about, not just from whichever terminal is open.
- **Mixed up IP addresses.** I confused which IP belonged to which machine. Keep a simple table (like the one above) while working.
- **Used placeholder usernames literally.** Examples use `user@...`; the real account name is required (and case-sensitive on Linux).
- **Firewall default-deny hid the problem.** An active firewall with no rules looks just like a broken SSH server. Check `ufw status` early.

## Security notes

- With password login off, the **private key is the only credential**. Protect it with a passphrase and full-disk encryption (for example FileVault on macOS).
- Never copy private keys between machines. Use one key pair per machine so a single one can be revoked without affecting the others.
- Restricting SSH to the local subnet in the firewall is an extra layer, not a replacement for key-only login.
- For remote access from outside the home network, use a VPN such as WireGuard instead of exposing port 22 to the internet.
- A laptop leaves the home network. Turn off Remote Login on macOS when it is not needed.
- Keep systems updated (`sudo pacman -Syu`, `sudo apt update && sudo apt upgrade`).

## Commands cheat sheet

| Task | Command |
|------|---------|
| Show IP addresses | `ip -br a` |
| Check sshd status | `systemctl status sshd` |
| Show listening ports | `ss -tlnp` |
| Generate key | `ssh-keygen -t ed25519` |
| Install public key | `ssh-copy-id user@host` |
| Verbose connection debug | `ssh -v user@host` |
| Force key-only login | `ssh -o PasswordAuthentication=no user@host` |
| Force password attempt | `ssh -o PubkeyAuthentication=no user@host` |
| Validate sshd config | `sudo sshd -t` |
| Show effective sshd settings | `sudo sshd -T` |
| Firewall status | `sudo ufw status verbose` |

## Skills practiced

Linux administration (Arch, Ubuntu), macOS command line, SSH public key authentication, sshd hardening, firewall rules with ufw, reading logs with journalctl, basic network troubleshooting.
