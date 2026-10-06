Site secrets
===

Site secrets are values a Jenkins deploy writes into a project's `.env`. The names live in local config. The values live in an Ansible vault. The Jenkins job decrypts that vault and inserts the values when it deploys.


Key names
---

`~/.config/base-docker/base-docker.ini` lists the names under `site_secret_keys`, as a comma-separated list:

```ini
site_secret_keys=foo,bar
```

Those names are the vault keys. The deploy uppercases them and replaces any character that is not a letter, number, or underscore, so `foo` becomes `FOO` in the `.env` file.


Write the values
---

```fish
bdcli site
```

This prompts once per name. Press enter on a prompt to keep the value already stored. The answers are written to `~/.config/base-docker/site_secrets.yml`, encrypted with the password in `~/.config/base-docker/pass.txt`. The same password encrypts `secrets.yml`.


Copy the vault onto the test host
---

```fish
bdcli deploy --env test-secrets
```

The playbook copies `site_secrets.yml` without decrypting it, to `/home/<user>/jenkins/site_secrets.yml` on the test host. The Jenkins container mounts that directory at `/var/jenkins_home`, so the job reads `/var/jenkins_home/site_secrets.yml`.


What the Jenkins job does
---

A job that runs `infra/deploy.yml` includes `infra/deploy-site-secrets.yml`. That task loads the vault and writes a block into the project `.env` (for a workspace job, `project/.env`):

```text
# BEGIN ANSIBLE MANAGED SECRETS
BAR="bar value"
FOO="foo value"
# END ANSIBLE MANAGED SECRETS
```

Lines outside the markers are left alone. The next deploy replaces the block in place.

The playbook is run with the Jenkins credential `site-passfile`:

```text
ansible-playbook ../deploy.yml --vault-password-file "$VAULT_PASS_FILE"
```

That one password has to decrypt both files the play loads: `infra/encrypted` and `site_secrets.yml`.


When the passwords differ
---

`bdcli site` encrypts `site_secrets.yml` with `pass.txt`. The job's `site-passfile` credential is the password that opens `infra/encrypted`. If those passwords differ, the play gets past the encrypted file and then fails on `load site secrets`. The task sets `no_log: true`, so the console shows `Action failed: Unknown error` instead of a decryption message.

Rekey the site file to the Jenkins password, then copy it up again. When rekey asks for the new password, use the `site-passfile` value:

```fish
ansible-vault rekey ~/.config/base-docker/site_secrets.yml --vault-password-file ~/.config/base-docker/pass.txt
bdcli deploy --env test-secrets
```

`bdcli site` still opens `site_secrets.yml` with `pass.txt`. After a rekey, later edits need `pass.txt` changed to that same Jenkins password, and `secrets.yml` rekeyed with it, because both local vaults are opened with `pass.txt`.
