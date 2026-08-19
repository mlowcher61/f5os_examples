# F5OS Examples

Sample Ansible playbooks for configuring **F5OS** — the operating system that runs
F5 **rSeries** appliances and **VELOS** chassis partitions — together with
everything needed to run them from **Red Hat Ansible Automation Platform (AAP)**.

Ten sample playbooks, a custom credential type, an inventory, an execution
environment and ten job templates, all created by config-as-code so you can
stand the whole thing up on a fresh AAP with a single command.

---

## Table of contents

- [What this repo gives you](#what-this-repo-gives-you)
- [The one concept you need to understand first](#the-one-concept-you-need-to-understand-first)
- [Repository layout](#repository-layout)
- [The playbooks](#the-playbooks)
- [Quick start on AAP](#quick-start-on-aap)
- [Running from a laptop instead](#running-from-a-laptop-instead)
- [The custom credential](#the-custom-credential)
- [Building the execution environment](#building-the-execution-environment)
- [Extending the repo](#extending-the-repo)
- [Safety notes](#safety-notes)
- [Troubleshooting](#troubleshooting)

---

## What this repo gives you

| Thing | Name in AAP | Created by |
|---|---|---|
| Custom credential type | `f5os_rseries` | `aap_config/01_credential_types.yml` |
| Credential | `F5OS rSeries` | `aap_config/02_credentials.yml` |
| Project | `F5OS Examples` | `aap_config/03_project.yml` |
| Inventory | `f5_inventory` | `aap_config/04_inventory.yml` |
| Execution environment | `F5OS EE` | `aap_config/05_execution_environment.yml` |
| Job templates | 10, one per playbook | `aap_config/06_job_templates.yml` |

Every job template is wired to the same project, inventory, execution environment
and credential. Pointing all ten at a different appliance is **one credential
change** — no code edits.

---

## The one concept you need to understand first

This is the only part of the design that isn't obvious, and everything else
follows from it.

**The `f5networks.f5os` modules have no `host`, `username` or `password`
arguments.** Unlike most modules, they don't take connection details at all.
They are *network* modules, so they read the connection from Ansible's
**httpapi connection plugin**. Practically, that means the appliance has to be a
real inventory host — you cannot reach it from a play running on `localhost`
with `connection: local`.

But we also want the appliance's address and password to live in an **AAP
credential**, not in git. Those two requirements pull in opposite directions.

The repo resolves it by splitting the connection into two halves:

```
inventory/group_vars/f5os.yml     the "f5os" GROUP holds HOW to talk to any
                                  F5OS box - protocol, port, SSL, timeouts.
                                  Identical for every appliance, not secret,
                                  safe in git.

F5OS rSeries credential (AAP)     holds WHICH box and WHOSE password -
                                  f5os_host, f5os_user, f5os_password.
                                  Per-device, secret, never in git.
```

and joining them at run time with a tiny role, `roles/f5os_target`. So **every
playbook here has exactly the same two-play shape**:

```yaml
# Play 1 - runs on localhost. Turns credential values into a real target.
- name: Build the F5OS target from the attached credential
  hosts: localhost
  connection: local
  roles:
    - f5os_target          # add_host into the "f5os" group

# Play 2 - runs against the appliance. This is where the work happens.
- name: Do the actual configuration
  hosts: f5os             # inherits the whole connection profile from group_vars
  tasks:
    - f5networks.f5os.f5os_vlan: ...
```

`add_host` puts the appliance into the `f5os` group, and it immediately inherits
that group's variables. The role only has to supply the three per-device values;
protocol, port and timeouts come from the group.

> **Why `inventory/group_vars/` and not `group_vars/` at the repo root?**
> Ansible only auto-loads a `group_vars/` directory that sits next to the
> **inventory file** or next to the **playbook**. A `group_vars/` at the repo
> root is silently ignored — the group would come up with no connection
> settings and every playbook would fail with a confusing error. This bit us
> while building the repo; the path is load-bearing.

---

## Repository layout

```
f5os_examples/
├── ansible.cfg                       # inventory path, long network timeouts
├── requirements.yml                  # collections, for LOCAL use only (see note)
│
├── inventory/
│   ├── hosts.yml                     # localhost + the empty f5os group
│   └── group_vars/
│       └── f5os.yml                  # <- the F5OS connection profile
│
├── roles/
│   └── f5os_target/                  # the credential -> inventory bridge
│
├── playbooks/                        # the 10 samples
│
├── execution-environment/            # ansible-builder definition for F5OS EE
│
└── aap_config/                       # config-as-code for AAP
    ├── deploy_aap.yml                # runs 00-06 in dependency order
    ├── 00_organization.yml           # ansible.platform
    ├── 01_credential_types.yml       # ansible.controller
    ├── 02_credentials.yml
    ├── 03_project.yml
    ├── 04_inventory.yml
    ├── 05_execution_environment.yml
    ├── 06_job_templates.yml
    └── vars/main.yml.example         # copy to main.yml (gitignored) and fill in
```

> **Note on `requirements.yml`:** it lives at the repo root on purpose. AAP
> automatically installs collections found at `collections/requirements.yml` on
> **every job run**, which adds a Galaxy round-trip to each job and fails on a
> disconnected controller. Because this solution ships a custom execution
> environment with the collections already baked in, the file is placed where
> AAP ignores it and only humans use it.

---

## The playbooks

Run **`F5OS - Device Info` first.** It is completely read-only and proves your
credential, execution environment and network path all work before anything
changes.

| # | Playbook / job template | What it does | Risk |
|---|---|---|---|
| 1 | **Device Info** | Reads interfaces, VLANs, tenants and system info | None — read-only |
| 2 | **Networking** | VLANs, physical interfaces, LACP LAGs | Low |
| 3 | **Tenant Deploy** | Imports a BIG-IP image, deploys a tenant, waits for its API | Medium |
| 4 | **Config Backup** | Backs up the config to a remote server | None — safe to schedule |
| 5 | **System Settings** | NTP, DNS, SNMP, syslog, management interface | Low (mgmt iface gated) |
| 6 | **Users and Auth** | Local users, roles, password policy, LDAP/RADIUS | Medium |
| 7 | **License** | Activates the platform licence | Low |
| 8 | **QKView** | Generates a diagnostic bundle for F5 Support | None |
| 9 | **Security Hardening** | Management TLS cert + IP allow-list | **High — can lock you out** |
| 10 | **System Upgrade** | Imports and installs an F5OS image | **High — reboots the appliance** |

Every playbook's variables are declared at the top of its second play with
working sample values and comments. Override them from the job template's extra
vars or a survey.

### A note on ordering

Two playbooks are written to be run in stages rather than in one shot:

- **Tenant Deploy** creates the tenant `configured` (defined but not running),
  then moves it to `deployed` in a second call. A bad size or VLAN reference
  therefore fails while the tenant is still inert, instead of half-way through a
  boot. The final wait uses `state: api-ready`, not `deployed` — "deployed" only
  means the VM started, and a follow-on BIG-IP job would fail against a tenant
  whose REST API isn't answering yet.
- **System Upgrade** separates import from install, so you can pull the image
  well before your change window and do only the reboot inside it.

---

## Quick start on AAP

**Prerequisites**

- AAP 2.5 or later, and an account that can create credentials, projects,
  inventories and job templates.
- An F5OS appliance (rSeries) reachable from your AAP execution nodes on the API
  port (8888 for RESTCONF, 443 for OpenAPI).
- A built and pushed execution environment — see
  [Building the execution environment](#building-the-execution-environment).
  (Or skip it for a first look; see the note in `05_execution_environment.yml`.)

**Steps**

```bash
git clone https://github.com/mlowcher61/f5os_examples.git
cd f5os_examples

# Collections needed to run the config-as-code from your machine
ansible-galaxy collection install -r requirements.yml

# Fill in your AAP and appliance details
cp aap_config/vars/main.yml.example aap_config/vars/main.yml
$EDITOR aap_config/vars/main.yml        # gitignored - never committed

# Create everything in AAP
ansible-playbook aap_config/deploy_aap.yml -e @aap_config/vars/main.yml
```

Then in the AAP UI, launch **F5OS - Device Info**. If it returns your
appliance's details, everything is wired correctly.

Re-running `deploy_aap.yml` is safe — every module is idempotent. You can also
run a single stage while iterating:

```bash
ansible-playbook aap_config/06_job_templates.yml -e @aap_config/vars/main.yml
```

### Why two collections in `aap_config/`

`00_organization.yml` uses **`ansible.platform`**; everything else uses
**`ansible.controller`**. That isn't inconsistency — it reflects how AAP 2.5 is
built. The organization is a *platform gateway* object shared by controller, EDA
and Automation Hub, so `ansible.platform` owns it. Credentials, projects,
inventories, execution environments and job templates are *controller* objects
and `ansible.controller` owns them.

> `ansible.platform` is certified content distributed through **Red Hat
> Automation Hub**. The copy on public Galaxy is a non-functional stub whose
> modules are literally documented as `Mocked` with an empty argument spec. It
> is already present in the `ee-supported-rhel9` base image, so jobs running on
> the F5OS EE have the real one.

---

## Running from a laptop instead

Everything works outside AAP too — useful for developing a new playbook.

```bash
ansible-galaxy collection install -r requirements.yml

export F5OS_HOST=10.10.10.5
export F5OS_USER=admin
export F5OS_PASSWORD='your-password'

ansible-playbook playbooks/f5os_device_info.yml
```

The `f5os_target` role's defaults read those environment variables. In AAP the
credential injects the same variables as extra vars, and **extra vars beat role
defaults**, so the identical playbook works in both places with no branching.

You can also pass them directly:

```bash
ansible-playbook playbooks/f5os_networking.yml \
  -e f5os_host=10.10.10.5 -e f5os_user=admin -e f5os_password='...'
```

---

## The custom credential

Credential type `f5os_rseries`, kind `net`:

| Field | Type | Required | Notes |
|---|---|---|---|
| `f5os_host` | string | yes | Management address of the appliance or partition |
| `f5os_user` | string | yes | Account with a sufficient F5OS role |
| `f5os_password` | string (secret) | yes | Write-only in the API, masked in job output |
| `f5os_port` | string | no | `8888` RESTCONF (default) or `443` OpenAPI |
| `f5os_validate_certs` | boolean | no | Enable after installing a trusted certificate |

**Injected as `extra_vars`, not environment variables.** The F5OS modules are
configured through Ansible *variables* (`ansible_host`, `ansible_user`,
`ansible_httpapi_password`), and the `f5os_target` role needs them as real
variables to hand to `add_host`. Environment variables would never reach the
connection plugin.

This follows the standard of keeping secrets in AAP rather than in vaulted files
in git: the password is write-only through the API, masked in output, and
governed by AAP's RBAC. Nothing about a specific appliance is ever committed.

**Adding a second appliance** — create another credential of the same type and
either swap it on the templates or copy a template and attach the new one.

---

## Building the execution environment

The stock *Default execution environment* does **not** contain
`f5networks.f5os`, so a custom EE is the right answer. It also makes jobs start
faster and behave identically every time, which matters for an upgrade job you
can't safely re-run.

```bash
cd execution-environment
ansible-builder build -t quay.io/mlowcher61/f5os-ee:latest -v3
podman push quay.io/mlowcher61/f5os-ee:latest
```

Set `ee_image` in `aap_config/vars/main.yml` to the image you pushed, then run
`aap_config/05_execution_environment.yml` to register it.

Base image is `registry.redhat.io/ansible-automation-platform-25/ee-supported-rhel9`,
which already carries the certified collections and a supported Python runtime.

**If you just want to try the repo without building an image**, set
`aap_execution_environment: "Default execution environment"` in your vars file
and add a `collections/requirements.yml` to the project so AAP installs
`f5networks.f5os` at job runtime. Slower and it fails on a disconnected
controller, but it works.

---

## Extending the repo

Adding an eleventh playbook takes two steps:

1. Create `playbooks/f5os_<thing>.yml` using the two-play shape above. Copy the
   first play verbatim from any existing playbook.
2. Add four lines to `f5os_job_templates` in `aap_config/06_job_templates.yml`
   and re-run that one file. The loop does the rest.

The full F5OS collection has 38 modules; this repo uses 19 of them. Others worth
knowing about: `f5os_fdb`, `f5os_lldp_config`, `f5os_stp_config`,
`f5os_primarykey`, `f5os_qos_*`, and the `velos_partition*` family for VELOS
chassis.

### VELOS

The `f5os_*` playbooks here run unchanged against a VELOS chassis partition —
point the credential at the partition instead of an rSeries appliance. Two
differences:

- Interface naming is `<blade>/<port>.0` (e.g. `2/1.0`) rather than `1.0`.
- Chassis partition lifecycle needs the `velos_partition*` modules, which this
  repo does not include.

---

## Safety notes

Two playbooks can cause real damage and are deliberately gated behind variables
that default to **off**:

**`f5os_security_hardening.yml`** — `f5os_allowed_ips` *replaces* the management
allow-list rather than appending to it. If your list omits the network your AAP
execution nodes are on, the job will succeed and you will never reach the
appliance again without console access. It refuses to run until you pass
`-e f5os_allowed_ips_reviewed=true`.

**`f5os_system_upgrade.yml`** — installing F5OS reboots the appliance and takes
every tenant on it down. Import runs by default; install requires
`-e f5os_perform_install=true`. Don't attach a schedule to this one; put it
behind an approval node in a workflow.

`f5os_system_settings.yml` can also cut your connection if you change the
management address, so its management-interface task is gated behind
`f5os_manage_mgmt_interface` (default `false`).

---

## Troubleshooting

**`No F5OS connection details were found`**
The credential isn't attached to the job template, or you didn't export the
environment variables locally. Check *Job Template → Credentials → F5OS rSeries*.

**`ansible_network_os is undefined`, or the appliance is unreachable from play 2**
The `f5os` group's variables weren't loaded. Confirm `inventory/group_vars/f5os.yml`
exists at that exact path (not at the repo root), and that `ansible.cfg` points
`inventory` at `inventory/hosts.yml`.

**`couldn't resolve module f5networks.f5os.f5os_vlan`**
The execution environment doesn't have the collection. Either build the custom
EE, or add `collections/requirements.yml` to the project.

**Timeouts on image import, tenant deploy or upgrade**
These are genuinely long operations. `ansible.cfg` sets a 1800-second persistent
connection timeout and `inventory/group_vars/f5os.yml` sets
`ansible_command_timeout: 1800`. Raise both if your image server is slow.

**An F5OS module fails with an unhelpful message**
`persistent_log_messages: true` is set in the group vars, so re-running the job
at verbosity 3 (`-vvv`, or *Verbosity: 3* on the template) puts the full
RESTCONF request and response in the output. That is almost always enough to see
what the appliance objected to.

**Certificate errors**
rSeries and VELOS ship self-signed certificates. `f5os_validate_certs` is `false`
by default. Turn it on after installing a trusted certificate with
`f5os_security_hardening.yml`.

---

## References

- [F5OS Ansible collection docs](https://clouddocs.f5.com/products/orchestration/ansible/devel/f5os/F5OS-index.html)
- [f5networks.f5os on the Red Hat certified catalog](https://catalog.redhat.com/en/software/collection/f5networks/f5os)
- [F5Networks/f5-ansible-f5os on GitHub](https://github.com/F5Networks/f5-ansible-f5os)
- [Automating F5OS on rSeries](https://clouddocs.f5.com/training/community/rseries-training/html/automating_rseries.html)

## Licence

MIT
