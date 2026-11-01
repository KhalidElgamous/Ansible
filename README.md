# Ansible

Ansible playbooks for deploying containerized services to Linux hosts, plus a Docker Compose stack that runs [Semaphore UI](https://semaphoreui.com/) (a web UI for Ansible) backed by PostgreSQL.

## What's in this repo

| Path | Purpose |
|------|---------|
| `deploy.yml` | Entry-point playbook. Runs the `deploy` role against all hosts in the inventory. |
| `roles/deploy/tasks/` | Tasks for the `deploy` role. |
| `deploy-portainer.yaml` | Playbook that deploys [Portainer CE](https://www.portainer.io/) as a Docker container. |
| `docker-compose.yml` | Semaphore UI + PostgreSQL 13, for running these playbooks from a web interface. |
| `config/` | Semaphore configuration, mounted into the container at `/etc/semaphore`. |

## Prerequisites

**Control node (where you run Ansible)**

- Ansible 2.12 or newer
- The `community.docker` collection:

  ```bash
  ansible-galaxy collection install community.docker
  ```

**Target hosts**

- SSH access with a user that can use `become`
- Docker installed and running
- Docker SDK for Python available to the Python interpreter Ansible uses on the host (required by the `community.docker` modules)
- For `deploy-portainer.yaml`: a RHEL/CentOS-style host with the `rh-python38` Software Collection installed at `/opt/rh/rh-python38`. The playbook enables it, adds it to `~/.bashrc`, and prints the resulting Python version.

## Inventory

No inventory is committed. Create one, for example `inventory/hosts`:

```ini
[servers]
server1 ansible_host=192.0.2.10 ansible_user=youruser
```

The Compose file mounts `./inventory/` into Semaphore as read-only, so keeping it in that folder works for both the CLI and the UI.

## Usage

### Run the `deploy` role

```bash
ansible-playbook -i inventory/hosts deploy.yml
```

### Deploy Portainer

```bash
ansible-playbook -i inventory/hosts deploy-portainer.yaml
```

This playbook:

1. Enables `rh-python38` and persists it in `~/.bashrc`
2. Creates a Docker volume named `portainer-data`
3. Starts the `portainer` container from `portainer/portainer-ce` with the Docker socket mounted and `restart_policy: unless-stopped`

Portainer is then available at `https://<host>:9443`. Create the admin user promptly after the first start.

Note: the playbook uses `become_user: "{{ lookup('env','USER') }}"`, which resolves to the user running Ansible on the control node. Make sure that user also exists on the targets, or change it.

## Semaphore UI

`docker-compose.yml` starts two containers:

- `postgres`: PostgreSQL 13 with a persistent volume
- `semaphoreui`: Semaphore `v2.8.90`, served on port `3000`

Setup:

```bash
mkdir -p inventory authorized-keys config

# Generate a key and put it in SEMAPHORE_ACCESS_KEY_ENCRYPTION in docker-compose.yml
head -c32 /dev/urandom | base64

docker compose up -d
```

Open <http://localhost:3000> and sign in with the admin account defined in the Compose file.

Mounted folders:

| Host path | Container path | Mode |
|-----------|----------------|------|
| `./inventory/` | `/inventory` | read-only |
| `./authorized-keys/` | `/authorized-keys` | read-only |
| `./config/` | `/etc/semaphore` | read-write |

## Security notes

Before using this anywhere beyond a local lab:

- Change the default admin and database passwords in `docker-compose.yml`.
- Move secrets out of the Compose file into a `.env` file (add it to `.gitignore`) and reference them as `${VARIABLE}`.
- Set `SEMAPHORE_ACCESS_KEY_ENCRYPTION` to a freshly generated value.
- `ANSIBLE_HOST_KEY_CHECKING=false` is set for convenience. Set it to `true` for production.
- Portainer gets full control of the Docker daemon through the mounted socket. Restrict access to port `9443` accordingly.

## Project layout

```text
.
├── config/
├── roles/
│   └── deploy/
│       └── tasks/
├── deploy.yml
├── deploy-portainer.yaml
├── docker-compose.yml
└── README.md
```

## License

No license has been specified yet. Add a `LICENSE` file if you want others to be able to reuse this.
