# Nexus Repository Manager — Production-Ready Ansible Role

Fully automated deployment, configuration, and lifecycle management of
**Sonatype Nexus Repository Manager OSS** on Ubuntu, using Ansible.

Installs Java, downloads and lays out Nexus, creates a dedicated system
user, configures the JVM/port, enables HTTPS, provisions blob stores and
repositories (Maven, Docker, npm), creates roles/users/privileges,
verifies health, and supports backup/restore — all idempotent and
tag-driven so you can run the whole thing or just one slice of it.

---

## Feature Checklist

| Area | Feature | Status |
|---|---|---|
| Core install | Install Java | ✅ |
| Core install | Create `/app` | ✅ |
| Core install | Download Nexus (only if missing) | ✅ |
| Core install | Extract Nexus (only once) | ✅ |
| Core install | Create `nexus` user | ✅ |
| Core install | Rename `nexus-*` → `nexus` | ✅ |
| Core install | Create `sonatype-work/nexus3` | ✅ |
| Core install | Set correct ownership | ✅ |
| Config | Configure `nexus.rc` | ✅ |
| Config | Configure `nexus.vmoptions` | ✅ |
| Service | Create systemd service | ✅ |
| Service | Reload systemd | ✅ |
| Service | Enable and start Nexus | ✅ |
| Verify | Wait for Nexus to become available | ✅ |
| Verify | Verify Nexus is running | ✅ |
| Dynamic | Dynamic version | ✅ |
| Dynamic | Dynamic port | ✅ |
| Dynamic | Dynamic JVM memory | ✅ |
| Security | Admin password automation | ✅ |
| Security | Enable anonymous access (variable) | ✅ |
| Security | SSL/HTTPS support | ✅ |
| Repos | Create blob store | ✅ |
| Repos | Create Maven hosted repository | ✅ |
| Repos | Create Docker hosted repository | ✅ |
| Repos | Create npm hosted repository | ✅ |
| RBAC | Create roles | ✅ |
| RBAC | Create users | ✅ |
| RBAC | Assign privileges | ✅ |
| Ops | Backup task | ✅ |
| Ops | Restore task | ✅ |
| Testing | Molecule tests | ✅ |
| CI | GitHub Actions | ✅ |
| Quality | Refactored, tagged, idempotent role | ✅ |
| Quality | Professional README | ✅ |
| Ops | Health check | ✅ |

---

## Project Structure

```text
nexus/
├── defaults/main.yml          # all tunable variables, sensible defaults
├── vars/main.yml               # internal, non-overridable computed vars
├── handlers/main.yml           # Reload systemd / Restart Nexus
├── meta/main.yml               # galaxy metadata
├── tasks/
│   ├── main.yml                 # orchestrator (import + tags)
│   ├── install.yml               # Java
│   ├── directories.yml           # /app + install-state detection
│   ├── download.yml               # fetch archive (idempotent)
│   ├── extract.yml                 # unpack + rename (first run only)
│   ├── user.yml                     # system user + ownership
│   ├── configure.yml                 # nexus.rc, vmoptions, port
│   ├── ssl.yml                        # HTTPS / keystore / jetty
│   ├── service.yml                     # systemd unit, enable/start
│   ├── wait.yml                         # wait_for + REST verify
│   ├── admin_password.yml                # password change + anonymous
│   ├── blobstores.yml                     # blob store provisioning
│   ├── repositories.yml                    # Maven / Docker / npm repos
│   ├── roles_users.yml                      # RBAC: roles, users, privileges
│   ├── healthcheck.yml                       # standalone health check
│   ├── backup.yml                             # stop, archive, prune, start
│   └── restore.yml                             # restore from archive
├── templates/
│   ├── nexus.rc.j2
│   ├── nexus.vmoptions.j2
│   ├── nexus.service.j2
│   ├── nexus-default.properties.j2
│   └── jetty-https.xml.j2
├── molecule/default/            # Molecule + Docker driver test scenario
│   ├── molecule.yml
│   ├── converge.yml
│   └── verify.yml
├── tests/
│   ├── inventory
│   └── test.yml
├── .github/workflows/ci.yml     # lint + Molecule in GitHub Actions
├── .yamllint
└── README.md
```

---

## Requirements

* Ubuntu 22.04 / 24.04
* Ansible ≥ 2.15
* Internet access to `download.sonatype.com`
* `sudo` privileges
* For SSL: `openssl` and `keytool` (bundled with the JDK this role installs)
* For Molecule tests: Docker, `molecule`, `molecule-plugins[docker]`

---

## Key Variables (`defaults/main.yml`)

| Variable | Purpose | Default |
|---|---|---|
| `nexus_version` | Nexus version to download | `3.83.1-03` |
| `nexus_port` | HTTP port | `8081` |
| `nexus_xms` / `nexus_xmx` / `nexus_direct_memory` | JVM heap / direct memory | `2g` / `2g` / `1g` |
| `nexus_admin_password` | Desired admin password (change on first boot) | `Admin@123456` |
| `nexus_anonymous_access` | Enable/disable anonymous read access | `false` |
| `nexus_blobstores` | List of file blob stores to create | `[{name: default, path: default}]` |
| `nexus_maven_repos` | List of Maven hosted repos | one `maven-releases` repo |
| `nexus_docker_repos` | List of Docker hosted repos | `[]` |
| `nexus_npm_repos` | List of npm hosted repos | `[]` |
| `nexus_roles` | List of roles with privileges | `[]` |
| `nexus_users` | List of users with role assignments | `[]` |
| `nexus_ssl_enabled` | Enable HTTPS | `false` |
| `nexus_ssl_cert_src` / `nexus_ssl_key_src` | PEM cert/key on control node | `""` |
| `nexus_backup_dir` | Where backups are stored | `/app/backups/nexus` |
| `nexus_backup_retention_days` | Prune backups older than this | `7` |

See the fully-commented `defaults/main.yml` for every option and example
list entries.

> **Secrets:** `nexus_admin_password`, user passwords, and the SSL
> keystore password should be supplied via `ansible-vault` in real
> environments rather than left at their defaults.

---

## Usage

### Full deployment

```bash
ansible-playbook -i tests/inventory tests/test.yml -K
```

### Run only part of the role (tags)

```bash
# Fresh install only
ansible-playbook tests/test.yml --tags install,configure,service -K

# (Re)configure repositories and RBAC without touching install/service
ansible-playbook tests/test.yml --tags repositories,security -K

# Enable HTTPS
ansible-playbook tests/test.yml --tags ssl,service \
  -e nexus_ssl_enabled=true \
  -e nexus_ssl_cert_src=/path/to/cert.pem \
  -e nexus_ssl_key_src=/path/to/key.pem -K

# Standalone health check
ansible-playbook tests/test.yml --tags healthcheck -K
```

### Backup

```bash
ansible-playbook tests/test.yml --tags backup -e nexus_action=backup -K
```

Creates a timestamped `tar.gz` of `sonatype-work` (config + blob store
data) under `nexus_backup_dir`, then prunes anything older than
`nexus_backup_retention_days`.

### Restore

```bash
ansible-playbook tests/test.yml --tags restore \
  -e nexus_action=restore \
  -e nexus_restore_archive=/app/backups/nexus/nexus-backup-XXXXXXXX.tar.gz -K
```

The existing `sonatype-work` is moved aside (not deleted) before the
archive is extracted, so a bad restore can be manually rolled back.

---

## Example: full-featured `test.yml`

```yaml
---
- hosts: localhost
  become: true
  vars:
    nexus_port: 8081
    nexus_admin_password: "{{ vault_nexus_admin_password }}"
    nexus_anonymous_access: false
    nexus_ssl_enabled: false

    nexus_docker_repos:
      - name: docker-hosted
        blob_store: default
        http_port: 8082

    nexus_npm_repos:
      - name: npm-hosted
        blob_store: default

    nexus_roles:
      - id: developers
        name: Developers
        privileges:
          - nx-repository-view-maven2-maven-releases-*
          - nx-repository-view-docker-docker-hosted-*

    nexus_users:
      - username: ci-bot
        password: "{{ vault_ci_bot_password }}"
        roles:
          - developers
  roles:
    - nexus
```

---

## Testing

### Molecule (local, Docker-based)

```bash
pip install molecule molecule-plugins[docker] ansible docker
molecule test
```

`molecule/default/converge.yml` applies the role on a disposable Ubuntu
24.04 container; `verify.yml` checks the systemd unit and REST status
endpoint.

### GitHub Actions

`.github/workflows/ci.yml` runs `yamllint` + `ansible-lint` on every
push/PR, then runs the full Molecule scenario.

---

## Verify Installation Manually

```bash
sudo systemctl status nexus
sudo journalctl -u nexus -f
sudo ss -tlnp | grep 8081
curl -u admin:<password> http://localhost:8081/service/rest/v1/status
```

Browser: `http://<SERVER_IP>:8081` (or `https://<SERVER_IP>:8443` if SSL
is enabled).

---

## Idempotency Notes

* Download/extract/rename only run when `nexus_home` doesn't already
  exist (`nexus_install.stat.exists` gate), so re-running the role is
  safe and fast on an already-provisioned host.
* Blob store, repository, role, and user creation calls tolerate `400`
  (already exists) alongside `201`/`200`/`204`, so re-runs don't fail.
* `admin_password.yml` checks whether the password was already changed
  before attempting to change it again.

---

## Future Improvements

* Reverse proxy (Nginx) + Let's Encrypt automation
* UFW firewall rules
* Cleanup policies via REST API
* LDAP/SAML realm configuration
* Multi-node / HA Nexus configuration

---

## Author

**Mohamed Adel**

* GitHub: https://github.com/DevNetMohamed
* LinkedIn: https://www.linkedin.com/in/mohammed-adel-406532281/

## License

MIT License.
