
BDCLI
===

[Base Docker](https://github.com/mikejw/base-docker) command line tool.

A wrapper CLI tool around base docker commands, including base docker installation
and initial configuration on a new machine.  (Base Docker dependencies still required - Python 3, virtualenv, Ansible, Docker.)

Currently no Windows support is available.

Check the [releases](https://github.com/mikejw/bdcli/releases) page!


Commands
---

These commands are available in the 1.6 release. That release is what exposes the platform features currently in development on Base Docker.

`bdcli` with no arguments, `help`, `-h`, or `--help` prints the short command list. `--version` prints the version.

Local app
---

| Command | What it does |
|---|---|
| `status` | Checks the running app. Requests `http://<host>/empathy/status`, using `host` from `~/.config/base-docker/base-docker.ini`, and prints `OK` or `Error!` |
| `dump` | Prints `base-docker.ini`, the local config that names the active project and host |
| `init` | First-time setup on a machine. Clones Base Docker, creates the virtualenv, and installs Ansible |
| `boot` | Starts the local dev containers for the project already selected in `base-docker.ini` |
| `switch` | Points the local environment at a different project and reconfigures it |
| `qs` | Creates a new project from a quickstart template and boots it |
| `cc` | Clears the running app's cache by running `empathy --clear_cache` in the app container |
| `cat` | Prints the newest Ansible playbook log from `~/.config/base-docker/logs`, which is the place to look after `init`, `boot`, `deploy`, or `qs` |
| `exec` | Opens an interactive shell in the tooling container, on this machine or over SSH if the dev host is remote. `--proxy` or `--test` instead opens a shell on that deployed host |
| `empathy` | Runs `php ./vendor/bin/empathy` in the app container. Anything after the command is passed through, for example `bdcli empathy --mysql populate` |
| `composer` | Runs `php ./composer.phar` in the app container. Anything after the command is passed through |
| `make` | Runs `make` in the app container. Anything after the command is passed through |
| `ant` | Runs `ant` in the app container. Anything after the command is passed through |

`init` options: `--dev`, `--h` (web host, when not using the default), `--cb` (project name).

`switch` options: `--cb` (project name).

`qs` options: `--cb` (project name), `--tpl` (`elib-cms`, `elib-blog`, `elib-acl`, or `elib-base`), `--bs5` (empathy core bootstrap 5 branch).

`exec --proxy` uses `hosts.proxy_user` and `hosts.proxy_ip` from `settings.yml`, and the `pem.proxy` private key from the Ansible vault. `exec --test` uses the matching `test_user`, `test_ip`, and `pem.test` values. The key is written to a temporary file for the SSH session and removed when the shell exits. Pass only one of `--proxy` or `--test`.


Settings and deploy
---

| Command | What it does |
|---|---|
| `site` | Prompts for each site secret listed in `base-docker.ini` and writes the answers into the encrypted `site_secrets.yml` |
| `secrets` | Same prompt flow for the shared Ansible vault, `~/.config/base-docker/secrets.yml`. Creates `pass.txt` the first time |
| `hosts` | Prompts for host addresses and writes them under `hosts` in `settings.yml`. This is where proxy and test IPs live |
| `reset` | Asks for confirmation, then deletes `pass.txt`, `secrets.yml`, and `settings.yml`. The vault password is gone with `pass.txt` |
| `deploy` | Runs the Ansible playbooks for one environment. `proxy` brings up the public Caddy host, `test` brings up the Jenkins host and records its Tailscale IP, and `test-secrets` only pushes secrets to the test host |
| `export` | Packs `settings.yml`, `secrets.yml`, `site_secrets.yml`, and `base-docker.ini` into a zip so the same local config can be moved to another machine |
| `pem` | Copies a private key file into the Ansible vault for a deployment environment. The same key is used for both proxy and test SSH |
| `keys` | Writes a new Ed25519 OpenSSH key pair under `/tmp/bd-key/<timestamp>/`. This is separate from the key the AWS stack generates |

`deploy` options: `--env` (`proxy`, `test`, or `test-secrets`), `--reset` (reset the test VM before boot).

`pem` options: `--env` (environment name), `--pem` (path to the private key).

`keys` options: `--private` and `--public` (file names). Both are required.


Platform
---

`platform` manages the AWS CloudFormation substrate in Base Docker. The AWS CLI must be installed, and the profile must already exist in `~/.aws`.

```text
bdcli platform --aws <up|down|status|key|update> [options]
```

| Action | What it does |
|---|---|
| `up` | Creates the VPC, proxy, and test host, waits until the stack is complete, saves the SSH key, and writes the new public IPs into `settings.yml`. If the stack is already up, it refreshes the key and IPs instead of creating a second one |
| `down` | Deletes that stack and everything it created: instances, addresses, VPC, and the key stored in SSM. The local PEM file is left in place |
| `status` | Prints whether the stack exists and, if it does, its outputs (public IPs, instance ids, key id) |
| `key` | Downloads the current stack private key from SSM again. Use this when the local PEM is missing but the stack is still up |
| `update` | Applies a changed template or parameter to the existing stack and waits for the update. Omitted parameters keep their current values |

Options: `--profile` (default `empathy`), `--region` (default `eu-west-2`), `--stack` (default `empathy-paas`), `--ssh-cidr` (on `up`, this machine's public IP `/32` when omitted), `--ami`, `--proxy-type`, `--test-type`, `--proxy-volume`, `--test-volume`, `--vpc-cidr`, `--subnet-cidr`.

`up` writes the key to `~/.config/base-docker/<stack>.pem`. `hosts.test_ts_ip` is not changed; Tailscale assigns that after the test host joins.
