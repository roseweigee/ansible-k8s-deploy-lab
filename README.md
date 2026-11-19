# Ansible deployment to Kubernetes — self-study lab

Deploys a small web app (Namespace, ServiceAccount, ConfigMap, Secret, Deployment, Service) to a Kubernetes
cluster with Ansible and the `kubernetes.core` collection, for two environments (dev and prod)
from one role.

What it practices:

- **Role + inventories**: one role (`roles/app`), two inventories (`inventories/dev`, `inventories/prod`)
  whose `group_vars` set namespace, replica count and page content.
- **Input validation**: the role asserts its variables first and fails with a clear message.
- **Secrets with ansible-vault**: prod reads the API token from an encrypted `vault.yml`
  (git-ignored); the Secret task runs with `no_log`.
- **Rollout on config change**: a checksum of the page content goes into the pod template,
  so a changed ConfigMap rolls the pods.
- **Idempotency**: a second run should report `changed=0`.
- **Preview and troubleshooting**: `--check --diff`, `-vvv`, and failure drills
  (bad image tag, bad variable, missing vault password).
- **HashiCorp Vault, two ways**: installs Vault (dev mode) and its Agent Injector with the Helm module,
  configures Kubernetes auth, a read-only policy and a role bound to the app's ServiceAccount, then
  deploys with `app_secret_mode=hashi_lookup` (Ansible reads Vault at deploy time) or
  `app_secret_mode=hashi_injector` (the pod fetches the token at start-up; no K8s Secret at all).
- **Verification as code**: `playbooks/verify.yml` checks ready replicas and fetches the page
  from inside a pod.
- **Lint**: passes `ansible-lint` with the `production` profile.

## Layout

```
ansible.cfg
requirements.yml                 kubernetes.core collection
inventories/
  dev/hosts.yml, group_vars/all/main.yml
  prod/hosts.yml, group_vars/all/main.yml, vault.example.yml
roles/app/
  defaults/main.yml              image, port, resources, timeouts
  tasks/main.yml                 assert → namespace → serviceaccount → configmap → secret → deployment (wait) → service
  templates/*.j2
playbooks/deploy.yml, verify.yml, teardown.yml
playbooks/vault-install.yml      HashiCorp Vault (dev mode) + Agent Injector via Helm
playbooks/vault-configure.yml    Kubernetes auth, policy, role, KV v2 secret
tests/render.yml                 offline: render every manifest and check it parses
```

## Quick start

Needs Python 3, Docker and a local cluster (kind).

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install ansible-core kubernetes hvac ansible-lint
ansible-galaxy collection install -r requirements.yml

ansible-playbook tests/render.yml                 # offline check, no cluster needed
ansible-playbook playbooks/deploy.yml             # dev
ansible-playbook playbooks/verify.yml
ansible-playbook playbooks/deploy.yml             # again: expect changed=0

# prod (2 replicas, token from ansible-vault)
cp inventories/prod/vault.example.yml inventories/prod/group_vars/all/vault.yml
ansible-vault encrypt inventories/prod/group_vars/all/vault.yml
ansible-playbook -i inventories/prod/hosts.yml playbooks/deploy.yml --ask-vault-pass
ansible-playbook -i inventories/prod/hosts.yml playbooks/verify.yml --ask-vault-pass

# HashiCorp Vault (needs helm)
ansible-playbook playbooks/vault-install.yml
kubectl port-forward -n vault svc/vault 8200:8200 &   # keep running
export VAULT_TOKEN=root
ansible-playbook playbooks/vault-configure.yml
ansible-playbook playbooks/deploy.yml -e app_secret_mode=hashi_injector
ansible-playbook playbooks/verify.yml -e app_secret_mode=hashi_injector

ansible-playbook playbooks/teardown.yml           # clean up dev
```

Set `app_k8s_context` (e.g. `-e app_k8s_context=kind-ansible-lab`) to target a specific kubeconfig context.

The step-by-step notes, failure drills and results are in [docs/NOTES.zh-TW.md](docs/NOTES.zh-TW.md) (Traditional Chinese).

## License

MIT

Vault runs in dev mode (in-memory, root token) for the lab only; production would need HA storage, auto-unseal, audit logging and backups.
