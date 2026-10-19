# ranch-server - Ansible Setup

One-command provisioning of a clean **Ubuntu 24.04** or **Debian 12** server with your toolbox.

**Always installed**
- Users: creates `cowboy` with passwordless sudo and your SSH public key.
- SSH: key-only, **port 9922** (`ssh_port` in `vars.yml` - the only place the port is defined).
- Containers: **Docker CE** (official Docker APT repo) + Buildx + **Compose v2**, with log rotation.
- Network: `nc`, `tcpdump`
- Build chain: `gcc`, `g++`, `build-essential`
- Updates: full upgrade on a fresh server only, then daily security updates (`unattended-upgrades`).
- Firewall: no host firewall - use your provider's firewall (see 3).

**Optional components** (all on by default; toggle in `vars.yml` or skip with `--skip-tags`)

| Variable | Tag | What |
|---|---|---|
| `install_apache` | `apache`, `lets-encrypt` | Apache with HTTPS-by-default + Let's Encrypt (best-effort). `index.html` is an "Under Maintenance" page (`files/maintenance.html`; `apache_maintenance_page: false` to manage it yourself), and the `Server` header / error pages show only `Apache` (no OS or version) |
| `install_wireguard` | `wireguard` | `wireguard`, `wireguard-tools` |
| `install_pyenv` | `pyenv` | pyenv `v2.8.10` + CPython `3.14.8` (system-wide, compiled - slow) |
| `install_wireshark` | `wireshark` | Wireshark + tshark; capture allowed for the `wireshark` group only |
| `install_mingw` | `mingw` | `mingw-w64` cross-compiler |
| `install_fail2ban` | `fail2ban` | fail2ban `sshd` jail on the SSH port |
| `swap_file_size_mb` | `swap` | `/swapfile` (default: 2 GB on servers with < 2 GB RAM and no swap) |

---

## 0) Repo layout

```
ranch-server/
├─ .github/workflows/ci.yml   # lint + full run in an Ubuntu 24.04 container
└─ ansible/
   ├─ ansible.cfg             # run everything from this directory
   ├─ inventory.example.ini   # copy to inventory.ini (git-ignored)
   ├─ vars.yml                # non-sensitive defaults: user, SSH port, toggles, versions
   ├─ site.yml                # the playbook
   ├─ files/.bashrc           # same file for root and cowboy (red prompt for root, green for users)
   ├─ templates/              # sshd_config, fail2ban jail
   └─ tests/                  # CI inventory and overrides
```

---

## 1) Prerequisites (on your laptop)

- **Ansible**: `brew install ansible` (macOS) or `sudo apt install -y ansible`.
  With `ansible-core` only, also run `ansible-galaxy collection install -r requirements.yml`.
- A server where `root` accepts your SSH key (e.g. a droplet created with your key).

---

## 2) Configure

All commands run from `ansible/` (that is where `ansible.cfg` lives).

```bash
cd ansible
cp inventory.example.ini inventory.ini   # IPs, hostname/domain, le_email, ssh_pubkey
```

`inventory.ini` is git-ignored: IPs, domains, e-mail and keys stay out of the repo.

---

## 3) Provider firewall

The playbook does not run a host firewall: ufw is disabled and iptables is flushed **on the
first run only** (later runs keep Docker's and fail2ban's chains). Note that Docker publishes
ports past ufw anyway, so filter at the provider. **Keep 22 open for the first run**, then
remove it. DigitalOcean example:

```bash
doctl compute firewall create --name ranch-fw \
  --droplet-ids <DROPLET_ID> \
  --inbound-rules "protocol:tcp,ports:22,address:0.0.0.0/0,address:::/0 protocol:tcp,ports:9922,address:0.0.0.0/0,address:::/0 protocol:tcp,ports:80,address:0.0.0.0/0,address:::/0 protocol:tcp,ports:443,address:0.0.0.0/0,address:::/0" \
  --outbound-rules "protocol:tcp,ports:all,address:0.0.0.0/0,address:::/0 protocol:udp,ports:all,address:0.0.0.0/0,address:::/0 protocol:icmp,address:0.0.0.0/0,address:::/0"

# after the first run:
doctl compute firewall remove-rules <FIREWALL_ID> \
  --inbound-rules "protocol:tcp,ports:22,address:0.0.0.0/0,address:::/0"
```

Add your WireGuard UDP port if you configure a tunnel.

---

## 4) Run

```bash
ansible-playbook site.yml
```

The same command works for the first run and every run after it:

1. The bootstrap play probes port **9922**. Closed -> fresh server -> connect as `root` on **22**.
2. Fresh server only: `apt dist-upgrade`, and a reboot if the upgrade asks for one. Later runs
   never upgrade or reboot, so re-running cannot take the site or a C2 session down.
3. Everything is installed and configured. Before moving sshd, the play **checks from your
   laptop that `cowboy` can log in with your key** (`ssh_login_check: false` skips it).
4. sshd moves to 9922 (on Ubuntu, `ssh.socket` activation is replaced by a plain
   `ssh.service`, so the port comes from `sshd_config`), and Ansible reconnects as `cowboy@9922`.

Log in with `ssh -p 9922 cowboy@<IP>`. Root login with a key still works (`PermitRootLogin
prohibit-password`).

---

## 5) Re-running, tags & idempotency

```bash
ansible-playbook site.yml                               # converge everything
ansible-playbook site.yml --check                       # preview
ansible-playbook site.yml -l 203.0.113.10               # one host
ansible-playbook site.yml --tags apache,lets-encrypt    # one component
ansible-playbook site.yml --skip-tags pyenv,wireshark   # skip heavy parts
```

A second run reports `changed=0` (checked in CI).

---

## 6) Component notes

**Apache + Let's Encrypt**
- The vhost name is `<hostname>.<domain>`; set `apache_server_name=...` on the host line to
  override it (older versions of this playbook used the bare `domain`).
- Let's Encrypt is attempted only when the host has a `domain` (or `apache_server_name`).
  `certbot certonly` obtains the certificate without touching the vhosts; renewals reload
  Apache through a deploy hook. If it fails (DNS not ready), the play prints certbot's error
  and Apache keeps the snakeoil certificate. Point DNS, then:
  `ansible-playbook site.yml --tags apache,lets-encrypt`
- HSTS (`max-age=63072000`, no `includeSubDomains`/`preload`, so `http://` callbacks to
  subdomains keep working) is sent only with a real certificate.

**Docker** - `/etc/docker/daemon.json` sets log rotation (`docker_log_max_size`,
`docker_log_max_file`) and `live-restore` (containers survive daemon restarts; not compatible
with Swarm). `docker_publish_localhost_only: true` makes `-p 8080:80` bind to 127.0.0.1; publish
on purpose with `-p 0.0.0.0:8080:80`.

**Hostname** - on cloud images, cloud-init is told to keep the hostname and `/etc/hosts`
(`/etc/cloud/cloud.cfg.d/99-ranch-server.cfg`), so a reboot does not bring the droplet name back.

**Wireshark** - capture is allowed for members of the `wireshark` group (`cowboy` is one),
not for every local user.

**Shell** - `files/.bashrc` is overwritten on every run. Put your own snippets in
`~/.bashrc.d/*.sh`; they are sourced and never touched.

**WireGuard** - only the packages are installed. To create a tunnel:

```bash
# on server (as root or sudo):
wg genkey | tee /etc/wireguard/server.key | wg pubkey > /etc/wireguard/server.pub
# create /etc/wireguard/wg0.conf with your desired config, then:
sudo systemctl enable --now wg-quick@wg0
```

---

## 7) Testing

CI (`.github/workflows/ci.yml`) runs `ansible-lint`, then the whole playbook twice inside an
Ubuntu 24.04 container that boots systemd, and fails if the second run changes anything. Same
thing locally:

```bash
docker run -d --name target --privileged --cgroupns=host \
  -v /sys/fs/cgroup:/sys/fs/cgroup:rw -v /var/lib/docker -v "$PWD":/repo:ro \
  geerlingguy/docker-ubuntu2404-ansible:latest
docker exec target bash -c "cd /repo/ansible && ansible-playbook -i tests/inventory.ci.ini \
  -e @tests/ci-vars.yml --skip-tags hostname site.yml"
```

---

## 8) Troubleshooting

**"Check that your key opens a session as cowboy" fails**
The key in `ssh_pubkey`/`ssh_pubkey_path` is not the one your SSH agent/`~/.ssh` offers.
Nothing in sshd was changed yet; fix the key and re-run.

**Ansible can't connect after the first run**
Port 9922 must be open in the provider firewall. On the server console:

```bash
sudo ss -ltnp | grep sshd
sudo systemctl status ssh
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication)'
```

**"Too many authentication failures"**
`MaxAuthTries 3`: with many keys in your agent, pin one with `-i ~/.ssh/key -o IdentitiesOnly=yes`.

---

## 9) Security reminders

* Keep SSH key-only access; the port is defined only in `vars.yml`.
* Keep IPs, domains and e-mail in `inventory.ini` (git-ignored). Older commits of this public
  repo still contain the previous `inventory.ini` and `vars.yml`.
* Filter traffic at the provider firewall.
