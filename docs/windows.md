# Windows setup

Windows support is available through **WSL 2, Ubuntu, and Docker**. Run bdcli
and its dependencies inside Ubuntu. This guide takes a fresh Windows setup
through installing the **v1.6.0-dev pre-release Linux binary** from GitHub.

## 1. Install WSL 2 and Ubuntu

Use Windows 11 or a Windows 10 version supported by
[Microsoft's WSL installation guide](https://learn.microsoft.com/en-us/windows/wsl/install).
Hardware virtualization must be enabled in your PC's BIOS/UEFI settings.

Open **PowerShell as Administrator** and run:

```powershell
wsl --install -d Ubuntu-24.04
```

Restart Windows when prompted. Open **Ubuntu 24.04 LTS** from the Start menu
and complete the first-run prompts to create your Linux username and password.
Use this normal Linux user for the remaining setup; `sudo` will ask for that
password when needed.

In PowerShell, update WSL and check the distribution version:

```powershell
wsl --update
wsl --list --verbose
```

The Ubuntu distribution should show version `2`. If it shows `1`, convert it:

```powershell
wsl --set-version Ubuntu-24.04 2
```

**Run all subsequent shell commands in the Ubuntu terminal**, except where
PowerShell is explicitly named. Keep projects in the Linux filesystem, for
example `~/code`, for Linux permissions and filesystem performance.

## 2. Install Python, virtualenv, and Ansible

```bash
sudo apt update
sudo apt install -y git curl ca-certificates python3 python3-pip python3-venv virtualenv ansible
mkdir -p ~/code

python3 --version
virtualenv --version
ansible --version
```

This installs the `virtualenv` command and Ansible in Ubuntu. There is no need
to create or activate a project virtualenv yet; `bdcli init` creates the Base
Docker virtualenv during its own setup.

## 3. Check systemd

Docker Engine will run as a service inside Ubuntu. Check that systemd is PID 1:

```bash
ps -p 1 -o comm=
```

If this prints `systemd`, continue to the next section. Otherwise, edit
`/etc/wsl.conf` with `sudo nano /etc/wsl.conf` and add or update this section,
preserving any other settings:

```ini
[boot]
systemd=true
```

Save the file, close Ubuntu, and run the following in **PowerShell**. This stops
all running WSL distributions, so save any work in them first:

```powershell
wsl --shutdown
```

Reopen Ubuntu and repeat the PID 1 check. See Microsoft's
[systemd instructions](https://learn.microsoft.com/en-us/windows/wsl/systemd)
if it is still unavailable.

## 4. Install Docker Engine in Ubuntu

These steps install Docker Engine directly in Ubuntu; Docker Desktop is not
required for this setup. If you already use Docker Desktop's WSL integration,
choose one Docker setup before continuing to avoid conflicting installations.
For an existing Docker installation, check the conflicting-package guidance in
the [official Ubuntu installation instructions](https://docs.docker.com/engine/install/ubuntu/).

Add Docker's signing key and package repository:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

### Allow your current user to run Docker

```bash
sudo usermod -aG docker "$USER"
```

The `docker` group grants root-level privileges; see Docker's
[post-installation guide](https://docs.docker.com/engine/install/linux-postinstall/).
Apply the new group membership to the current shell:

```bash
newgrp docker
```

Verify Docker works **without `sudo`**:

```bash
docker run --rm hello-world
docker compose version
```

If you get a socket permission error, close your Ubuntu sessions and reopen
Ubuntu to refresh group membership, then retry. If the daemon is unavailable,
run `sudo systemctl start docker` and check `systemctl status docker`.

## 5. Install bdcli from GitHub

The current version is **v1.6.0-dev**, a **pre-release**. Open the
[v1.6.0-dev release](https://github.com/mikejw/bdcli/releases/tag/v1.6.0-dev)
and expand **Assets**. Use the Linux binary, even though your host is Windows.
Do not select GitHub's source-code archives.

Check your Ubuntu architecture:

```bash
uname -m
```

Choose the matching Linux asset: `x86_64` corresponds to x64/amd64, and
`aarch64` corresponds to ARM64. Only use an asset built for your architecture.

Copy the asset's download link from GitHub. In Ubuntu, replace the placeholder
below with that link, keeping the quotes:

```bash
curl -fL 'PASTE_LINUX_BINARY_DOWNLOAD_URL_HERE' -o /tmp/bdcli
```

If the release asset is compressed, download and extract the archive instead,
then use the extracted `bdcli` executable as the source in the install command.
For a directly downloaded binary:

```bash
sudo install -m 0755 /tmp/bdcli /usr/local/bin/bdcli
bdcli --version
bdcli --help
```

Check that the reported version matches the `v1.6.0-dev` release. Ubuntu now has
Python, virtualenv, Ansible, Docker configured for your user, and bdcli installed.
Continue with the [README command reference](../README.md) when ready to run
`bdcli init`. The AWS CLI is an additional prerequisite only for the platform
commands.
