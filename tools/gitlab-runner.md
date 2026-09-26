# GitLab shell runner flow

Variables that are used here and require change:
- `<RUNNER_TAG>`: the tag by which the runner is going to be triggered, like `agrobank-oxus`
- `<RUNNER_TOKEN>`: the `glrt-`-prefixed token GitLab shows once the runner is created
- `<DEPLOY_TOKEN_USERNAME>`: username of the deploy token used in step 3

Deploys are done by a tagged GitLab shell runner installed on the application server.

## 1. Installation

Create a **group-scoped** GitLab runner in the UI and provide the following settings:
- tags: `<RUNNER_TAG>`
- keep "Run untagged jobs" unticked
- description: "Agrobank environment shell runner"
- `shell` executor

Once you have your **token**, install the package and register the runner.
Tags and executor are already set in the UI, so registration only requires the token:

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install -y gitlab-runner
```

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url https://gitlab.com/ \
  --token "<RUNNER_TOKEN>" \
  --executor shell
```

## 2. Granting rights

The pipeline pulls the new image, rewrites `TAG` in the app's tag file under `/etc/<APP>/`,
and restarts the unit.

Thus, it needs particular rights.
Execute the commands below to grant them.

Add `gitlab-runner` user to the `docker` group:
```bash
sudo usermod -aG docker gitlab-runner
```

The paths below must match the image tag files created in the application docs.

Sudoers rules in `/etc/sudoers.d/gitlab-runner-deploy-backend`:
```bash
gitlab-runner ALL=(root) NOPASSWD: /usr/bin/sed -i s|^TAG=.*|TAG=agrobank-*| /etc/oxus-backend/agrobank.env
gitlab-runner ALL=(root) NOPASSWD: /usr/bin/systemctl restart agrobank-app-oxus-backend
```

Sudoers rules in `/etc/sudoers.d/gitlab-runner-deploy-frontend`:
```bash
gitlab-runner ALL=(root) NOPASSWD: /usr/bin/sed -i s|^TAG=.*|TAG=agrobank-*| /etc/oxus-frontend/agrobank.env
gitlab-runner ALL=(root) NOPASSWD: /usr/bin/systemctl restart agrobank-app-oxus-frontend
```

Sudoers rules in `/etc/sudoers.d/gitlab-runner-deploy-models`:
```bash
gitlab-runner ALL=(root) NOPASSWD: /usr/bin/sed -i s|^TAG=.*|TAG=agrobank-*| /etc/oxus-models-api/agrobank.env
gitlab-runner ALL=(root) NOPASSWD: /usr/bin/systemctl restart agrobank-app-oxus-models-api
```

Sudoers rules in `/etc/sudoers.d/gitlab-runner-deploy-prefect`:
```bash
gitlab-runner ALL=(root) NOPASSWD: /usr/bin/sed -i s|^TAG=.*|TAG=agrobank-*| /etc/oxus-prefect/worker.env
gitlab-runner ALL=(root) NOPASSWD: /usr/bin/sed -i s|^TAG=.*|TAG=agrobank-*| /etc/oxus-prefect/server.env
gitlab-runner ALL=(root) NOPASSWD: /usr/bin/systemctl restart agrobank-app-oxus-prefect-worker
gitlab-runner ALL=(root) NOPASSWD: /usr/bin/systemctl restart agrobank-app-oxus-prefect-server
```

Grant the correct rights, verify every file, and restart the runner to pick up the `docker`
group:
```bash
chmod 440 /etc/sudoers.d/gitlab-runner-deploy-*
visudo -cf /etc/sudoers.d/gitlab-runner-deploy-backend
visudo -cf /etc/sudoers.d/gitlab-runner-deploy-frontend
visudo -cf /etc/sudoers.d/gitlab-runner-deploy-models
visudo -cf /etc/sudoers.d/gitlab-runner-deploy-prefect
systemctl restart gitlab-runner
```

## 3. Log in to the Container Registry

This login is for the systemd-unit, not the runner. 

The **deploy token** had already been prepared for Agrobank inside current operating Vault.

You will need to ask for it directly, it's under the following path in the UI:

`secrets -> amudario -> tools -> gitlab -> agrobank-deploy-token`

Now you can **log in** to the registry, as a root user.
You can use `read -rs` and `--password-stdin` to keep the token out of shell history.

```bash
read -rs DT
docker login registry.gitlab.com -u '<DEPLOY_TOKEN_USERNAME>' --password-stdin <<<"$DT"
unset DT
```

Make sure "`Login succeeded`" pops up on your terminal.