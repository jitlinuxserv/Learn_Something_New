Yes. You can configure **3-step SSH authentication** exactly as follows:

> **1. SSH Public Key → 2. Linux Password → 3. Google Authenticator OTP → Login**

The important point is that **Password + Google Authenticator OTP will both be handled by PAM through `keyboard-interactive`**.

# Final Authentication Flow

```text
SSH Connection on Port 4832
          │
          ▼
1. Public Key Authentication
          │
       SUCCESS
          │
          ▼
2. Linux Password
          │
       SUCCESS
          │
          ▼
3. Google Authenticator OTP
          │
       SUCCESS
          │
          ▼
       LOGIN
```

---

# STEP 1 — Keep Your Current SSH Session Open

⚠️ **Very important**

Do not close your current SSH session until you test everything from a second terminal.

---

# STEP 2 — Install Google Authenticator PAM Module

On Ubuntu/Debian:

```bash
sudo apt update
sudo apt install libpam-google-authenticator
```

Verify:

```bash
dpkg -l | grep google-authenticator
```

---

# STEP 3 — Configure Google Authenticator for Your User

⚠️ Run this command as the **actual SSH user**, not with `sudo`.

For example, if you login as `ubuntu`:

```bash
google-authenticator
```

It will ask questions.

Recommended answers:

```text
Do you want authentication tokens to be time-based (y/n)
```

Answer:

```text
y
```

You will see a QR code.

📱 Open the **Google Authenticator app** on your phone:

1. Tap `+`
2. Scan a QR code
3. Scan the QR code displayed on the server

It will also show:

```text
Your new secret key is:
Your verification code is:
Your emergency scratch codes are:
```

⚠️ **Save the emergency scratch codes somewhere secure.**

Then answer:

```text
Do you want me to update your "~/.google_authenticator" file?
```

```text
y
```

For rate limiting:

```text
Do you want to disallow multiple uses of the same authentication token?
```

Recommended:

```text
y
```

For time window:

```text
By default, tokens are good for 30 seconds...
```

Recommended:

```text
y
```

For rate limiting:

```text
Do you want to enable rate-limiting?
```

Recommended:

```text
y
```

---

# STEP 4 — Verify Google Authenticator File

Run:

```bash
ls -la ~/.google_authenticator
```

Set secure permissions:

```bash
chmod 600 ~/.google_authenticator
```

Check:

```bash
ls -l ~/.google_authenticator
```

Expected:

```text
-rw------- 1 username username ... .google_authenticator
```

---

# STEP 5 — Configure PAM

Edit:

```bash
sudo nano /etc/pam.d/sshd
```

You need both:

1. Linux password authentication
2. Google Authenticator OTP

The order is important.

Add these lines:

```text
# First ask Linux password
auth required pam_unix.so

# Then ask Google Authenticator OTP
auth required pam_google_authenticator.so
```

## Recommended placement

Your `/etc/pam.d/sshd` should contain the PAM authentication configuration in this order:

```text
# Standard Unix authentication - Password
@include common-auth

# Google Authenticator - OTP
auth required pam_google_authenticator.so
```

### Recommended Ubuntu Configuration

On Ubuntu, I recommend using:

```text
@include common-auth
auth required pam_google_authenticator.so
```

Do **not** add both:

```text
auth required pam_unix.so
```

and:

```text
@include common-auth
```

because `common-auth` already handles the normal Linux password authentication.

---

# STEP 6 — Configure SSH Authentication

Create/edit:

```bash
sudo nano /etc/ssh/sshd_config.d/99-custom-auth.conf
```

Use:

```conf
# =========================================
# SSH Authentication
# =========================================

# Public Key Authentication
PubkeyAuthentication yes

# Keyboard Interactive Authentication
KbdInteractiveAuthentication yes

# Enable PAM
UsePAM yes

# Disable normal SSH password authentication
PasswordAuthentication no

# Require Public Key FIRST
# Then Keyboard Interactive through PAM
AuthenticationMethods publickey,keyboard-interactive

# Security
PermitRootLogin no
PermitEmptyPasswords no
```

---

# Why This Configuration Works

Your SSH configuration:

```conf
AuthenticationMethods publickey,keyboard-interactive
```

means:

```text
FIRST:

Public Key
```

Then:

```text
SECOND:

Keyboard Interactive
```

The Keyboard Interactive authentication goes to PAM:

```text
keyboard-interactive
        │
        ▼
       PAM
        │
        ▼
Linux Password
        │
        ▼
Google Authenticator OTP
```

Therefore, the complete flow is:

```text
Public Key
    +
Linux Password
    +
Google Authenticator OTP
```

---

# STEP 7 — Configure SSH Port 4832 Using ssh.socket

Create directory:

```bash
sudo mkdir -p /etc/systemd/system/ssh.socket.d
```

Create:

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

# STEP 8 — Test SSH Configuration

Before restarting SSH:

```bash
sudo sshd -t
```

If there is no output:

```text
Configuration is valid
```

Check effective configuration:

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

# STEP 9 — Reload systemd

```bash
sudo systemctl daemon-reload
```

Restart the socket:

```bash
sudo systemctl restart ssh.socket
```

Check:

```bash
sudo systemctl status ssh.socket
```

---

# STEP 10 — Verify Port 4832

Run:

```bash
sudo ss -tlnp | grep 4832
```

You should see SSH listening on:

```text
*:4832
```

---

# STEP 11 — Configure UFW

Allow the new port:

```bash
sudo ufw allow 4832/tcp
```

Check:

```bash
sudo ufw status
```

---

# STEP 12 — Oracle Cloud Firewall

If this Oracle VM is running on Oracle Cloud Infrastructure, also allow:

```text
Protocol: TCP

Port: 4832

Source:
YOUR_PUBLIC_IP/32
```

For temporary testing:

```text
0.0.0.0/0
```

But production should preferably use your own public IP.

---

# STEP 13 — Test From a NEW Terminal

⚠️ Keep your old SSH session open.

Connect:

```bash
ssh -p 4832 username@SERVER_IP
```

Example:

```bash
ssh -p 4832 ubuntu@YOUR_SERVER_IP
```

You should experience something similar to:

```text
1. Public key authentication
   ↓
   SUCCESS

2. Password:
   Enter your Linux user password
   ↓
   SUCCESS

3. Verification code:
   Enter Google Authenticator OTP
   ↓
   SUCCESS

4. LOGIN
```

---

# Complete Final Configuration

## 1. SSH Configuration

### `/etc/ssh/sshd_config.d/99-custom-auth.conf`

```conf
# Public Key Authentication
PubkeyAuthentication yes

# Keyboard Interactive Authentication
KbdInteractiveAuthentication yes

# Enable PAM
UsePAM yes

# Disable direct password authentication
PasswordAuthentication no

# Require Public Key first
# Then PAM Keyboard Interactive
AuthenticationMethods publickey,keyboard-interactive

# Security
PermitRootLogin no
PermitEmptyPasswords no
```

---

## 2. PAM Configuration

### `/etc/pam.d/sshd`

Ensure the authentication order is:

```text
# Linux Password
@include common-auth

# Google Authenticator OTP
auth required pam_google_authenticator.so
```

### Authentication sequence:

```text
PAM
 │
 ├── Linux Password
 │
 └── Google Authenticator OTP
```

---

## 3. SSH Socket

### `/etc/systemd/system/ssh.socket.d/listen.conf`

```ini
[Socket]
ListenStream=
ListenStream=4832
```

---

# Complete Architecture

```text
                     INTERNET
                        │
                        │ TCP 4832
                        ▼
               OCI Firewall / NSG
                        │
                        ▼
                     UFW
                        │
                        ▼
                 ssh.socket
                        │
                        ▼
                     sshd
                        │
                        ▼
          ┌─────────────────────────┐
          │  STEP 1                 │
          │  SSH Public Key         │
          └───────────┬─────────────┘
                      │ SUCCESS
                      ▼
          ┌─────────────────────────┐
          │  STEP 2                 │
          │  Linux Password         │
          │  PAM common-auth        │
          └───────────┬─────────────┘
                      │ SUCCESS
                      ▼
          ┌─────────────────────────┐
          │  STEP 3                 │
          │  Google Authenticator   │
          │  OTP                    │
          └───────────┬─────────────┘
                      │ SUCCESS
                      ▼
                   LOGIN
```

# Important: Test Before Closing Your Session

I strongly recommend testing with verbose mode:

```bash
ssh -vvv -p 4832 username@SERVER_IP
```

On the server, monitor logs:

```bash
sudo tail -f /var/log/auth.log
```

Or:

```bash
sudo journalctl -f -u ssh.service
```

---

## Final Result

Your Oracle/Ubuntu VM will require:

| Step | Authentication           |
| ---- | ------------------------ |
| 1️⃣  | SSH Public Key           |
| 2️⃣  | Linux User Password      |
| 3️⃣  | Google Authenticator OTP |
| 4️⃣  | SSH Login                |

And SSH will listen on:

```text
TCP 4832
```

**Important:** Before applying the PAM change, make sure Google Authenticator is configured for at least your SSH user. A PAM misconfiguration can lock you out, so keep the existing session open and test from a second terminal.
