# Change SSH Port from 22 to 4832 with `ssh.socket`

## Require **Public Key + Password/Keyboard-Interactive Authentication**

This guide is for **Ubuntu/Debian systems using systemd socket activation (`ssh.socket`)**, including Oracle VM environments running Ubuntu.

Your requested authentication setup:

```text
AuthenticationMethods publickey,keyboard-interactive
```

This means:

1. User must authenticate with their **SSH public key**
2. Then they must complete **keyboard-interactive authentication** (normally PAM password)

Both are required. This is **2-factor authentication**, not "either key or password". ([Debian Manpages][1])

---

# 1. Important: Keep Your Current SSH Session Open

⚠️ **Do not close your existing SSH connection** until you successfully test port `4832`.

Open a **second terminal** for testing.

---

# 2. Check Whether Your System Uses `ssh.socket`

Run:

```bash
sudo systemctl status ssh.socket
```

If it shows:

```text
active (listening)
```

then continue with this guide.

Also check:

```bash
sudo systemctl is-enabled ssh.socket
```

---

# 3. Configure SSH Authentication

Create a separate SSH configuration file:

```bash
sudo nano /etc/ssh/sshd_config.d/99-custom-auth.conf
```

Add:

```conf
# Enable Public Key Authentication
PubkeyAuthentication yes

# Enable Keyboard Interactive Authentication
KbdInteractiveAuthentication yes

# Enable PAM
UsePAM yes

# Disable normal password authentication
PasswordAuthentication no

# Require Public Key + Keyboard Interactive
AuthenticationMethods publickey,keyboard-interactive

# Recommended security settings
PermitRootLogin no
PermitEmptyPasswords no
```

Save the file.

### Why `PasswordAuthentication no`?

Because you specifically requested:

```conf
AuthenticationMethods publickey,keyboard-interactive
```

The password prompt will normally be handled through:

```text
keyboard-interactive → PAM → Linux password
```

So the login flow becomes:

```text
SSH Client
    │
    ▼
Public Key Authentication
    │
    │ SUCCESS
    ▼
Keyboard Interactive
    │
    ▼
PAM Password Prompt
    │
    ▼
LOGIN SUCCESS
```

OpenSSH supports requiring multiple authentication methods in sequence. ([Debian Manpages][1])

---

# 4. Check Your SSH Configuration

Before restarting anything:

```bash
sudo sshd -t
```

If there is **no output**, the configuration syntax is valid.

Check the effective authentication configuration:

```bash
sudo sshd -T | grep -Ei 'pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|usepam|authenticationmethods'
```

Expected:

```text
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication yes
usepam yes
authenticationmethods publickey,keyboard-interactive
```

---

# 5. Change SSH Port to 4832 Using `ssh.socket`

Because you want to use **`ssh.socket`**, configure the systemd socket.

Create the override directory:

```bash
sudo mkdir -p /etc/systemd/system/ssh.socket.d
```

Create the override file:

```bash
sudo nano /etc/systemd/system/ssh.socket.d/listen.conf
```

Add exactly:

```ini
[Socket]
ListenStream=
ListenStream=4832
```

### Important

The empty:

```ini
ListenStream=
```

clears the existing socket configuration.

Then:

```ini
ListenStream=4832
```

sets the new SSH port.

This is the standard systemd override approach for socket-activated SSH. ([Apuntes técnicos][2])

---

# 6. Reload systemd

Run:

```bash
sudo systemctl daemon-reload
```

---

# 7. Restart `ssh.socket`

Run:

```bash
sudo systemctl restart ssh.socket
```

Check status:

```bash
sudo systemctl status ssh.socket
```

You should see that it is listening on port `4832`.

---

# 8. Verify SSH is Listening on Port 4832

Run:

```bash
sudo ss -tulpn | grep 4832
```

You should see something similar to:

```text
LISTEN 0 4096 *:4832
```

Also verify port 22 is no longer listening:

```bash
sudo ss -tulpn | grep ':22'
```

If everything is correct, there should be no SSH listener on port `22`.

You can inspect the complete socket configuration with:

```bash
sudo systemctl cat ssh.socket
```

And check the active settings:

```bash
sudo systemctl show ssh.socket -p Listen
```

---

# 9. Enable Firewall Port 4832

## If using UFW

Check:

```bash
sudo ufw status
```

Allow the new port:

```bash
sudo ufw allow 4832/tcp
```

Verify:

```bash
sudo ufw status
```

You should see:

```text
4832/tcp ALLOW
```

⚠️ **Do not remove port 22 firewall access until port 4832 has been successfully tested.**

After successful testing:

```bash
sudo ufw delete allow 22/tcp
```

---

# 10. If Using Oracle Cloud Infrastructure (OCI)

You must also allow port **4832** in the Oracle Cloud network configuration.

Allow:

```text
Protocol: TCP
Source: Your Public IP
Destination Port: 4832
```

For testing, you can temporarily allow:

```text
0.0.0.0/0
```

But for production, restrict it to:

```text
YOUR_PUBLIC_IP/32
```

Also check whether your Oracle VM has:

* OCI Network Security Group (NSG)
* Security List
* Network firewall
* Ubuntu UFW

All relevant layers must allow TCP port `4832`.

---

# 11. Test From a NEW Terminal

Keep your existing SSH session open.

Test:

```bash
ssh -p 4832 username@SERVER_IP
```

Example:

```bash
ssh -p 4832 ubuntu@203.0.113.10
```

Expected authentication flow:

```text
Public Key Authentication
        ↓
Success
        ↓
Password Prompt / Keyboard Interactive
        ↓
Enter Linux Password
        ↓
Login Successful
```

For debugging:

```bash
ssh -vvv -p 4832 username@SERVER_IP
```

---

# 12. Verify Public Key is Installed

On the server:

```bash
ls -la ~/.ssh
```

Check:

```bash
cat ~/.ssh/authorized_keys
```

Permissions should be:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Ownership:

```bash
chown -R $USER:$USER ~/.ssh
```

For a specific user:

```bash
sudo chown -R username:username /home/username/.ssh
```

---

# 13. Test Authentication Methods

Check SSH logs while connecting.

## Ubuntu

```bash
sudo journalctl -u ssh.service -f
```

Or:

```bash
sudo journalctl -u ssh.socket -f
```

Also:

```bash
sudo tail -f /var/log/auth.log
```

You should see public key authentication followed by keyboard-interactive authentication.

---

# Complete Recommended Configuration

## `/etc/ssh/sshd_config.d/99-custom-auth.conf`

```conf
# ==========================================
# SSH Authentication Configuration
# ==========================================

# Enable Public Key Authentication
PubkeyAuthentication yes

# Enable Keyboard Interactive Authentication
KbdInteractiveAuthentication yes

# PAM handles keyboard-interactive authentication
UsePAM yes

# Disable direct password authentication
PasswordAuthentication no

# Require BOTH:
# 1. Public Key
# 2. Keyboard Interactive (PAM Password)
AuthenticationMethods publickey,keyboard-interactive

# Security
PermitRootLogin no
PermitEmptyPasswords no
```

---

## `/etc/systemd/system/ssh.socket.d/listen.conf`

```ini
[Socket]
ListenStream=
ListenStream=4832
```

---

# Complete Command Sequence

Here is the clean setup in order.

### Step 1 — Create SSH authentication configuration

```bash
sudo nano /etc/ssh/sshd_config.d/99-custom-auth.conf
```

Add:

```conf
PubkeyAuthentication yes
KbdInteractiveAuthentication yes
UsePAM yes
PasswordAuthentication no
AuthenticationMethods publickey,keyboard-interactive
PermitRootLogin no
PermitEmptyPasswords no
```

---

### Step 2 — Test configuration

```bash
sudo sshd -t
```

---

### Step 3 — Create socket override directory

```bash
sudo mkdir -p /etc/systemd/system/ssh.socket.d
```

---

### Step 4 — Create socket configuration

```bash
sudo nano /etc/systemd/system/ssh.socket.d/listen.conf
```

Add:

```ini
[Socket]
ListenStream=
ListenStream=4832
```

---

### Step 5 — Reload systemd

```bash
sudo systemctl daemon-reload
```

---

### Step 6 — Restart SSH socket

```bash
sudo systemctl restart ssh.socket
```

---

### Step 7 — Verify port

```bash
sudo ss -tulpn | grep 4832
```

---

### Step 8 — Open firewall

```bash
sudo ufw allow 4832/tcp
```

---

### Step 9 — Test configuration from another terminal

```bash
ssh -p 4832 username@SERVER_IP
```

---

# Verify Everything

Run:

```bash
sudo sshd -T | grep -Ei 'port|pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|usepam|authenticationmethods'
```

And:

```bash
sudo systemctl status ssh.socket
```

And:

```bash
sudo ss -tlnp | grep 4832
```

---

# Final Architecture

```text
                    Internet
                       │
                       │ TCP 4832
                       ▼
              OCI Security List / NSG
                       │
                       ▼
                 Oracle VM
                       │
                       ▼
                    UFW
                 TCP 4832
                       │
                       ▼
                systemd ssh.socket
                       │
                       ▼
                    sshd
                       │
                       ▼
              Public Key Required
                       │
                       ▼
                 Key Success
                       │
                       ▼
         Keyboard Interactive Required
                       │
                       ▼
                 PAM Password
                       │
                       ▼
                 SSH Login
```

## Important distinction

Your configuration:

```conf
AuthenticationMethods publickey,keyboard-interactive
```

means **BOTH authentication methods are mandatory**.

If instead you want:

> Login using Public Key **OR** Password

then use:

```conf
PubkeyAuthentication yes
PasswordAuthentication yes
KbdInteractiveAuthentication yes
UsePAM yes
```

and **do not set** `AuthenticationMethods`.

For your requested setup, the recommended configuration is:

```conf
AuthenticationMethods publickey,keyboard-interactive
```

which provides **Public Key + Password through PAM keyboard-interactive**. ([Debian Manpages][1])

[1]: https://manpages.debian.org/trixie/openssh-server/sshd_config.5.en.html?utm_source=chatgpt.com "sshd_config(5) — openssh-server — Debian trixie — Debian Manpages"
[2]: https://luispa.com/en/posts/2023-04-14-ssh-socket/?utm_source=chatgpt.com "Socketed SSH | Technical Notes"
