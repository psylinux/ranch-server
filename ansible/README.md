# ranch-server · Ansible Setup

One-command provisioning of a clean Debian server with your toolbox:

- Tools: `nc`, `wireshark` + `tshark`, `tcpdump`, **ntopng** (via Docker), `apache2` (HTTPS by default), Let’s Encrypt (best-effort), **Docker CE + Buildx + Compose v2**, `pyenv` + latest CPython, `gcc`/`g++`, `mingw-w64`, `git`, `curl`, `jq`, `unzip`, and **WireGuard**.
- Users: creates `cowboy` with passwordless sudo and your SSH public key.
- SSH hardening (opt-in tag): port **2222**, keys only, `AllowUsers cowboy`, `MaxAuthTries 3`, no root SSH.

> This repo assumes **Debian** on the remote host and **Ubuntu/macOS/Linux** on your laptop.

---

## 0) Repo layout

```

ranch-server/
├─ ansible/
│  ├─ inventory.ini         # where your server lives
│  ├─ vars.yml              # your variables (user, domain, email, key)
│  └─ site.yml              # the playbook
├─ cloud-init/              # (optional) original cloud-init files
└─ README.md                # this file

````

---

## 1) Prerequisites (on your laptop)

- **Ansible**
  - Ubuntu/Debian: `sudo apt update && sudo apt install -y ansible`
  - macOS (Homebrew): `brew install ansible`
- **sshpass** (only for the **first run** if you must use a password)
  - Ubuntu/Debian: `sudo apt install -y sshpass`
  - macOS (Homebrew taps): `brew install hudochenkov/sshpass/sshpass` or `brew install esolitos/ipa/sshpass`

---

## 2) Configure your target

### 2.1 Inventory
Edit `ansible/inventory.ini` with your server’s IP and initial SSH user/port:

```ini
[ranch]
YOUR.SERVER.IP.ADDR ansible_user=root ansible_port=22
````

> After hardening, you’ll switch this to `cowboy` on port `2222` (see step 6).

### 2.2 Variables

Put your values in `ansible/vars.yml`:

```yaml
primary_user: cowboy
ssh_pubkey: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQD...USU= you@host"
domain: "evilbuffer.com"
le_email: "you@example.com"
```

* `ssh_pubkey` is your **public** key (e.g., from `~/.ssh/id_ed25519.pub`).
* LE issuance is **best-effort** (will succeed once DNS points to your server).

---

## 3) First run (safe phase)

The first run:

* installs Python on the remote if needed,
* installs Docker (official repo), WireGuard, toolchains, Apache, etc.,
* creates `cowboy` and adds your SSH key,
* **skips SSH hardening** to avoid lockouts.

From repo root or `ansible/`:

```bash
cd ansible
ansible-playbook -i inventory.ini site.yml \
  --skip-tags ssh_hardening \
  -k \
  --ssh-common-args='-o StrictHostKeyChecking=accept-new'
```

* `-k` asks for the **SSH password** (requires `sshpass` on your laptop).
* If you already copied your key to **root**, you can omit `-k`.

**Verify** a few things:

```bash
ansible -i inventory.ini ranch -m command -a "docker --version"
ansible -i inventory.ini ranch -m command -a "tshark --version | head -1"
ansible -i inventory.ini ranch -m command -a "ss -ltnp | egrep ':80|:443'"
ansible -i inventory.ini ranch -m command -a "docker ps --format '{{.Names}} {{.Ports}}'"
```

**ntopng** UI (Docker): `http://YOUR.SERVER.IP.ADDR:3000/`

---

## 4) Logging in as `cowboy`

After the first run, your key is on `cowboy`:

```bash
ssh cowboy@YOUR.SERVER.IP.ADDR  # still on port 22 (hardening not applied yet)
```

If that works, proceed to hardening.

---

## 5) Apply SSH hardening (opt-in)

This switches SSH to **port 2222**, keys only, no root SSH, `AllowUsers cowboy`.

```bash
ansible-playbook -i inventory.ini site.yml --tags ssh_hardening
```

Now update the inventory to reflect the new user/port:

```ini
[ranch]
YOUR.SERVER.IP.ADDR ansible_user=cowboy ansible_port=2222
```

Reconnect:

```bash
ssh -p 2222 cowboy@YOUR.SERVER.IP.ADDR
```

> **Lockout recovery**: if you misconfigure SSH, use your provider’s web console (OOB) to edit `/etc/ssh/sshd_config` and restart `ssh`.

---

## 6) Re-running & idempotency

Re-run anytime; tasks only change what’s needed:

```bash
ansible-playbook -i ansible/inventory.ini ansible/site.yml
```

Preview changes (after you’ve run once successfully):

```bash
ansible-playbook -i ansible/inventory.ini ansible/site.yml --check
```

---

## 7) How to add more servers

Append more lines to `inventory.ini`:

```ini
[ranch]
IP1 ansible_user=cowboy ansible_port=2222
IP2 ansible_user=cowboy ansible_port=2222
```

Target all at once (parallelized), or a subset with `-l`:

```bash
ansible-playbook -i ansible/inventory.ini ansible/site.yml -l IP1
```

---

## 8) WireGuard notes (what gets installed)

The playbook installs `wireguard` and `wireguard-tools`. You can then create peers:

```bash
# on server (as root or sudo):
wg genkey | tee /etc/wireguard/server.key | wg pubkey > /etc/wireguard/server.pub
# create /etc/wireguard/wg0.conf with your desired config, then:
sudo systemctl enable --now wg-quick@wg0
```

If you want the playbook to manage full WireGuard configs, open an issue or request a patch; I’ll add a templated role.

---

## 9) ntopng (Docker) management

* Compose file: `/opt/ntopng/docker-compose.yml`
* Data dir: `/opt/ntopng/data`

Common commands:

```bash
sudo docker compose -f /opt/ntopng/docker-compose.yml ps
sudo docker compose -f /opt/ntopng/docker-compose.yml logs -f
sudo docker compose -f /opt/ntopng/docker-compose.yml pull
sudo docker compose -f /opt/ntopng/docker-compose.yml up -d
```

---

## 10) Troubleshooting

**Ansible can’t connect (port 22 refused)**
SSH might be listening on another port or blocked by firewall. Use provider console:

```bash
sudo systemctl status ssh
sudo ss -ltnp | grep ssh
```

**“to use the 'ssh' connection type with passwords … install sshpass”**
Install `sshpass` on your **laptop** and re-run with `-k`.

**Missing APT packages (e.g., `ntopng`, `software-properties-common`)**

* We run ntopng via Docker to avoid distro repo differences.
* The playbook already removed `software-properties-common`.
  If you hit another missing package, open an issue; we’ll patch the play.

**Let’s Encrypt fails**
DNS isn’t pointing yet. The play keeps Apache HTTPS using snakeoil certs. Point DNS, then re-run:

```bash
ansible-playbook -i ansible/inventory.ini ansible/site.yml --tags apache,lets-encrypt
```

**Python missing on remote**
The first play installs it via a `raw` task. If needed, do it manually once:

```bash
apt-get update -y && apt-get install -y python3 python3-apt sudo
```

---

## 11) Using patches (deltas)

When you receive a `*.patch` file:

**With Git (recommended):**

```bash
cd ~/github/ranch-server
git apply --check updates.patch      # dry-run
git apply --whitespace=fix updates.patch
git add -A && git commit -m "Apply updates.patch"
```

**Without Git:**

```bash
cd ~/github/ranch-server
patch -p1 < updates.patch
```

**Undo**:

```bash
git apply -R updates.patch
# or with patch(1):
patch -R -p1 < updates.patch
```

---

## 12) Security reminders

* Keep SSH key-only access; use the hardening tag asap.
* Restrict firewall to needed ports (22 or 2222, 80/443, 3000 if you expose ntopng).
* Prefer pinning your repo to trusted commits and review changes.

---

## 13) Common commands (cheat sheet)

```bash
# First-time run (password once)
ansible-playbook -i ansible/inventory.ini ansible/site.yml \
  --skip-tags ssh_hardening -k \
  --ssh-common-args='-o StrictHostKeyChecking=accept-new'

# Apply SSH hardening
ansible-playbook -i ansible/inventory.ini ansible/site.yml --tags ssh_hardening

# Day-2 re-run
ansible-playbook -i ansible/inventory.ini ansible/site.yml

# Dry-run preview
ansible-playbook -i ansible/inventory.ini ansible/site.yml --check

# Quick checks
ansible -i ansible/inventory.ini ranch -m ping
ansible -i ansible/inventory.ini ranch -m command -a "docker --version"
ansible -i ansible/inventory.ini ranch -m command -a "sshd -T | egrep 'port|maxauthtries|permitrootlogin|passwordauthentication|allowusers'"
```

---

## 14) What gets installed/configured (summary)

* **Docker CE** + Buildx + Compose v2 (official Docker APT repo)
* **WireGuard** (`wireguard`, `wireguard-tools`)
* Network tools: `nc`, `wireshark`, `tshark`, `tcpdump`, **ntopng (Docker)**
* Web: `apache2` with HTTPS-by-default, optional **Let’s Encrypt**
* Python: **pyenv** + latest CPython (system-wide)
* Build toolchain: `gcc`, `g++`, `mingw-w64`, `build-essential`, libs for CPython builds
* User: **cowboy**, passwordless sudo, your `ssh_pubkey`
* **SSH hardening** (opt-in tag): 2222, keys only, `AllowUsers cowboy`, `MaxAuthTries 3`, no root SSH
