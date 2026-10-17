# Ansible-VPS

Ansible playbooks that harden an IONOS VPS (Ubuntu 22.04/24.04 or Debian 12/13) and run
[Hermes Agent](https://github.com/NousResearch/hermes-agent) and
[AIOStreams](https://github.com/Viren070/AIOStreams) in Docker, each as its own
locked-down Linux user.

## What it does

| Role | What it sets up |
|------|-----------------|
| `common` | Full upgrade, daily automatic security updates, kernel network/info-leak hardening |
| `ssh` | Key-only login, no root, only `admin_user` may log in (refuses to run if you have no key installed) |
| `firewall` | UFW: deny everything inbound except 22 (rate limited), 80, 443. fail2ban bans SSH brute forcers |
| `docker` | Docker Engine from Docker's repo, log rotation, `live-restore`, no inter-container traffic on the default bridge |
| `caddy` | HTTPS reverse proxy with automatic Let's Encrypt certs, the only thing on 80/443 |
| `hermes` | `hermes` system user (uid 10001), `/srv/hermes`, gateway + dashboard containers |
| `aiostreams` | `aiostreams` system user (uid 10002), `/srv/aiostreams`, container with no capabilities, port bound to localhost |

### How the users keep the apps apart

- `hermes` and `aiostreams` have no password, no shell, and are **not** in the `docker`
  group (being in that group is the same as being root).
- Each app can write only to its own `/srv/<app>/data` (mode `0700`). The compose file and
  the secrets file (`<app>.env`, mode `0600`) belong to root, so the app can't edit its
  own setup.
- The containers run as those UIDs, so a compromised AIOStreams can't read Hermes' memory
  or API keys, and the reverse holds too.
- Only you (`admin_user`, with your sudo password) can change any of it.

## Network layout

```
internet ──443──▶ Caddy ──▶ 127.0.0.1:3000 ▶ AIOStreams
internet ──22───▶ sshd (key only, you only)
                   └─ ssh -L tunnel ▶ 127.0.0.1:9119 ▶ Hermes dashboard
Hermes gateway: outbound only (LLM APIs, Telegram, etc.)
```

## First-time setup

### 1. On your Mac

```sh
brew install ansible            # or: uv tool install ansible
ansible-galaxy collection install -r requirements.yml
ls ~/.ssh/id_ed25519.pub || ssh-keygen -t ed25519   # your SSH key
```

### 2. Before touching the server

- **DNS:** create an `A` record for your AIOStreams domain pointing at the VPS IP.
  Caddy can't get a certificate until this resolves.
- **IONOS firewall:** in the IONOS panel (Network > Firewall Policies), allow only
  TCP 22, 80, 443 (and UDP 443). That's a second firewall in front of UFW.

### 3. Fill in your settings

- `inventory/hosts.yml`: your VPS IP.
- `group_vars/all/vars.yml`: `admin_user`, `aiostreams_domain`, `acme_email`.
- Secrets:

  ```sh
  cp vault.yml.example group_vars/all/vault.yml
  # fill it in (comments explain how to generate each value), then:
  ansible-vault encrypt group_vars/all/vault.yml
  ```

  The encrypted file is safe to commit. Don't lose the vault password.

### 4. Bootstrap (once, as root)

```sh
ssh-copy-id root@<VPS_IP>                         # uses the IONOS root password one last time
ansible-playbook bootstrap.yml -u root --ask-vault-pass
ssh <admin_user>@<VPS_IP>                                # check this works before going on
```

### 5. Run the main playbook

```sh
ansible-playbook site.yml -K -J                    # -K = sudo password, -J = vault password
```

Later, `--check --diff` does a dry run that shows what *would* change. (On a fresh server
it stops at the Docker step, since a dry run doesn't really install Docker.)

Keep your existing SSH session open during the first run. If anything goes wrong you
can still fix it from there.

After this, `root` can't log in at all and passwords are rejected over SSH.

## Day to day

| Task | Command |
|------|---------|
| Re-apply everything (safe, only fixes drift) | `ansible-playbook site.yml -K -J` |
| Only one part | `ansible-playbook site.yml -K -J --tags aiostreams` |
| Update an app | bump `aiostreams_image` / `hermes_image` in `vars.yml`, re-run with that tag |
| Change a secret | `ansible-vault edit group_vars/all/vault.yml`, re-run |
| Hermes dashboard | `ssh -L 9119:localhost:9119 <admin_user>@<VPS_IP>` then open http://localhost:9119 |
| Hermes interactive setup | `sudo docker exec -it hermes hermes model` |
| Logs | `sudo docker logs -f aiostreams` / `sudo docker logs -f hermes` |
| Ban list | `sudo fail2ban-client status sshd` |

AIOStreams is at `https://<aiostreams_domain>`. Log in to its dashboard with the
`vault_aiostreams_auth` credentials.

## Hermes security notes

Hermes is an agent: it runs shell commands. Inside its container it runs as `hermes`, sees
only `/srv/hermes/data`, and has no Docker socket, so it can't control other containers.
On top of that:

- Set `TELEGRAM_ALLOWED_USERS` (or the equivalent for your chat platform) so strangers can't
  command your bot.
- Keep command approval on: `sudo docker exec -it hermes hermes config set approval_mode ask`.
- Use a provider API key with a spending limit.
- It uses host networking (as upstream does) so the dashboard can bind to the VPS's own
  localhost. This means it can reach other localhost services, such as AIOStreams on :3000.
