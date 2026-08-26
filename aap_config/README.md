# aap_config — config-as-code for Ansible Automation Platform

Creates every AAP object this solution needs, so a new user can stand the whole
thing up on a fresh controller with one command.

```bash
cp vars/main.yml.example vars/main.yml
$EDITOR vars/main.yml                       # gitignored - never committed
ansible-playbook aap_config/deploy_aap.yml -e @aap_config/vars/main.yml

or if running from a bastion host instead of the controller:

ansible-playbook aap_config/deploy_aap.yml \
  -e @aap_config/vars/main.yml \
  -e "aap_hostname=https://aap.mycompany.com"
```

Everything is idempotent. Re-run it after any change.

## Prerequisites on the machine you run this from

You need the two collections from the repo-root `requirements.yml`, plus the
Python `requests` library — **installed for the interpreter that runs
`ansible-playbook`, which is often not the one `pip3` points at.**

```bash
ansible-galaxy collection install -r ../requirements.yml

# Find the interpreter ansible actually uses, then install into THAT one.
ansible --version | grep -i 'python version'
sudo dnf install -y python3.12-requests        # match your version
python3.12 -c 'import requests'                # must print nothing
```

On RHEL 9 `pip3` is Python 3.9's pip while ansible-core is built against Python
3.12, so `pip3 install requests` reports success and fixes nothing. Only
`00_organization.yml` needs `requests`, because `ansible.platform` ships action
plugins that run inside the `ansible-playbook` process on the control node
rather than as ordinary modules. `00_preflight.yml` checks this for you and
fails with a readable message instead of a multiprocessing traceback.

None of this applies to jobs launched from AAP itself — the execution
environment in `execution-environment/` already includes these.

## Files, in dependency order

| File | Creates | Collection |
|---|---|---|
| `00_preflight.yml` | Nothing — asserts control node prerequisites | `ansible.builtin` |
| `00_organization.yml` | Organization | `ansible.platform` |
| `01_credential_types.yml` | `f5os_rseries` credential type | `ansible.controller` |
| `02_credentials.yml` | `F5OS rSeries` credential (+ optional SCM credential) | `ansible.controller` |
| `03_execution_environment.yml` | `F5OS EE` (+ optional registry credential) | `ansible.controller` |
| `04_project.yml` | `F5OS Examples` project | `ansible.controller` |
| `05_inventory.yml` | `f5_inventory`, `localhost` host, `f5os` group | `ansible.controller` |
| `06_job_templates.yml` | 10 job templates | `ansible.controller` |

The numbering is dependency order, not preference: a credential needs its type,
a job template needs the project, inventory, EE and credential to exist first.
`deploy_aap.yml` just imports them in sequence.

Run a single stage while iterating:

```bash
ansible-playbook aap_config/06_job_templates.yml -e @aap_config/vars/main.yml
```

## Why two collections

`ansible.platform` is preferred wherever it applies, but it only applies to
**platform gateway** objects. In AAP 2.5 the organization is a gateway object
shared across controller, EDA and Automation Hub, so `ansible.platform` owns it.
Credentials, projects, inventories, execution environments and job templates are
**controller** objects, so `ansible.controller` owns those. That split is a
property of the platform, not a style choice.

Note the auth parameters differ between the two:

- `ansible.controller` → `controller_host`, `controller_username`, `controller_password`, `validate_certs`
- `ansible.platform` → `aap_hostname`, `aap_username`, `aap_password`, `aap_validate_certs`

Each playbook sets these once via `module_defaults` on its action group
(`group/ansible.controller.controller` and `group/ansible.platform.gateway`)
rather than repeating them on every task. `00_organization.yml` falls back to
the `controller_*` values if you don't set the `aap_*` ones, so most people only
need to fill in one set.

> `ansible.platform` must come from **Red Hat Automation Hub**. The public
> Galaxy build is a non-functional stub — its modules are documented as `Mocked`
> with an empty argument spec. The real collection is already inside the
> `ee-supported-rhel9` base image used by the F5OS EE.

## The inventory has a deliberate shape

`05_inventory.yml` creates a `localhost` host **and** an `f5os` group. Appliances
are never listed. Instead:

- `localhost` runs the first play of every playbook, with `ansible_connection: local`.
- The `f5os` group carries the shared httpapi connection profile as group
  variables, and appliances are added to it at run time by the `f5os_target`
  role using values from the credential.

The group's variables are **not written out in this playbook**. They are read
directly from `inventory/group_vars/f5os.yml` in the repo:

```yaml
f5os_group_vars: "{{ lookup('ansible.builtin.file', playbook_dir ~ '/../inventory/group_vars/f5os.yml') | from_yaml }}"
```

That keeps a single source of truth — the file a developer edits on a laptop and
the group AAP uses cannot drift apart. Change the connection profile in one
place and re-run `05_inventory.yml`.

## Secrets

`vars/main.yml` is gitignored; only `vars/main.yml.example` is committed. It is
used **only to build** the AAP objects. Once the credential exists in AAP, the
running job templates get their secrets from AAP, not from that file — so you can
delete it afterwards.

If you'd rather the password never touch disk at all, create the credential with
a placeholder and set the real one once in the AAP UI. AAP never returns a
secret through its API, so a later re-run of `02_credentials.yml` will not
overwrite it unless you supply a new value.
