# ranch-server - Ansible Setup

One-command provisioning of a clean Debian server with your toolbox:

**Tools installed**
- Network: `nc`, `wireshark`, `tshark`, `tcpdump`
- Web: `apache2` with HTTPS-by-default, Let’s Encrypt (best-effort)
- Containers: **Docker CE** (official Docker APT repo) + Buildx + **Compose v2**
- Python: **pyenv** + latest CPython (system-wide)
- Build chain: `gcc`, `g++`, `mingw-w64`, `build-essential` and CPython build deps
- VPN: **WireGuard** (`wireguard`, `wireguard-tools`)

- Users: creates `cowboy` with passwordless sudo and your SSH public key.
- SSH runs on port **22** (root used for the first run/password; user switches to `cowboy` after).
- Firewall: host firewall is disabled/emptied (ufw stopped + iptables/ip6tables flushed). Open ports 80/443/22 in the Vultr panel.

> This repo assumes:
> - **Debian** on the remote host.
> - **Ubuntu/macOS/Linux** on your laptop.

---

## 0) Repo layout

```

ranch-server/
├─ ansible/
│  ├─ inventory.ini         # where your server lives
│  ├─ vars.yml              # your variables (user, domain, email, key).
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
> Use `root`/22 for the first run (`-k`), then switch inventory to `cowboy` on port 22 after your key is installed.

### 2.2 Variables

Put your values in `ansible/vars.yml`:

```yaml
primary_user: cowboy
ssh_pubkey: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQD...USU= you@host"
domain: "evilbuffer.com"
le_email: "you@example.com"
```

**Tip (recommended):** On the first run, override `ssh_pubkey` with your *actual* local key without editing files:

```bash
-e ssh_pubkey_path=~/.ssh/id_ed25519.pub
```

* `ssh_pubkey` is your **public** key (e.g., from `~/.ssh/id_ed25519.pub`).
* LE issuance is **best-effort** (will succeed once DNS points to your server).

---

## 3) First run (safe phase)

The first run:

* installs Python on the remote if needed,
* installs Docker (official repo), WireGuard, toolchains, Apache, etc.,
* creates `cowboy` and adds your SSH key,
* sets up SSH with your key (root SSH disabled after install).

From repo root or `ansible/`:

```bash
cd ansible
ansible-playbook -i inventory.ini site.yml \
  -k \
  -e ssh_pubkey_path=~/.ssh/id_ed25519.pub \
  --ssh-common-args='-o StrictHostKeyChecking=accept-new'
```

* `-k` asks for the **SSH password** (requires `sshpass` on your laptop).
* If you already copied your key to **root**, you can omit `-k`.
* Passing `-e ssh_pubkey_path=...` guarantees the correct key lands on `cowboy` even if `vars.yml` contains an older key.

**Verify** a few things:

```bash
ansible -i inventory.ini ranch -m command -a "docker --version"
ansible -i inventory.ini ranch -m command -a "tshark --version | head -1"
ansible -i inventory.ini ranch -m command -a "ss -ltnp | egrep ':80|:443'"
ansible -i inventory.ini ranch -m command -a "docker ps --format '{{.Names}}'"
```

Tip: you can also pass the key inline (careful with shell quoting):
`-e "ssh_pubkey=$(cat ~/.ssh/id_ed25519.pub)"`, but `ssh_pubkey_path` is safer.

---

## 4) Re-running & idempotency

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
IP1 ansible_user=cowboy ansible_port=22
IP2 ansible_user=cowboy ansible_port=22
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

## 9) Troubleshooting

**Ansible can’t connect (port 22 refused)**
SSH might be listening on another port or blocked by your provider-level firewall. Use the provider console (or disable rules in Vultr’s panel) and check:

```bash
sudo systemctl status ssh
sudo ss -ltnp | grep ssh
```

**“to use the 'ssh' connection type with passwords … install sshpass”**
Install `sshpass` on your **laptop** and re-run with `-k`.

**Missing APT packages**
If you hit a missing package on your image, open an issue; we’ll patch the play for your Debian release.

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

## 10) Security reminders

* Keep SSH key-only access.
* Manage network rules in the Vultr panel (playbook does **not** touch iptables/nftables). Open 80/443/22 there as needed.
* Prefer pinning your repo to trusted commits and review changes.

---

## 11) Common commands (cheat sheet)

```bash
# First-time run (password once, installs your key)
ansible-playbook -i ansible/inventory.ini ansible/site.yml \
  -k \
  -e ssh_pubkey_path=~/.ssh/id_ed25519.pub \
  --ssh-common-args='-o StrictHostKeyChecking=accept-new'

# Re-run / day-2
ansible-playbook -i ansible/inventory.ini ansible/site.yml

# Dry-run preview
ansible-playbook -i ansible/inventory.ini ansible/site.yml --check

# Quick checks
ansible -i ansible/inventory.ini ranch -m ping
ansible -i ansible/inventory.ini ranch -m command -a "docker --version"
ansible -i ansible/inventory.ini ranch -m command -a "sshd -T | egrep 'port|maxauthtries|permitrootlogin|passwordauthentication|allowusers'"
```

---

## After applying

Re-run to converge:

```bash
cd ~/github/ranch-server/ansible
ansible-playbook -i inventory.ini site.yml \
  --ssh-common-args='-o StrictHostKeyChecking=accept-new'
````
