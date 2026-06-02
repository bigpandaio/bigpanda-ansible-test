# bigpanda-ansible-test

Test playbooks and Job Template recipes for the
[`bigpanda.incident`](https://github.com/bigpandaio/bigpanda-ansible) Ansible
Collection.

This repo is **not** the collection itself — it's a tiny consumer project
that demonstrates how to drive the collection's modules from
[Ansible Automation Platform](https://www.redhat.com/en/technologies/management/ansible)
(AAP). Use it as the **Project** in an AAP Controller, attach a Job Template
to one of the playbooks, and you can run the BigPanda actions from the AAP UI
with a Survey for the inputs.

---

## What's in here

```
.
├── collections/
│   └── requirements.yml          # tells AAP to install bigpanda.incident from Automation Hub
└── playbooks/
    ├── smoke_comment.yml         # 1 task: bigpanda.incident.comment      (safe)
    ├── add_tag.yml               # 1 task: bigpanda.incident.add_tag      (safe)
    ├── assign.yml                # 1 task: bigpanda.incident.assign       (safe)
    ├── resolve_alert.yml         # 1 task: bigpanda.incident.resolve_alert  (destructive)
    ├── split.yml                 # 1 task: bigpanda.incident.split        (destructive)
    ├── merge.yml                 # 1 task: bigpanda.incident.merge        (destructive)
    └── resolve.yml               # 1 task: bigpanda.incident.resolve      (destructive — closes incident)
```

Each playbook is intentionally minimal: one task that calls one module, with
every parameter sourced from `extra_vars` (which AAP populates from a Survey).

---

## Quick start

### Prerequisites

1. An **AAP instance** with Automation Controller and a local Private
   Automation Hub. The fastest free option is the
   [Red Hat Developer Sandbox](https://developers.redhat.com/developer-sandbox)
   — it includes AAP 2.5 with Controller + EDA + Hub.
2. The `bigpanda.incident` collection (v1.3.0 or newer) **uploaded to your
   Automation Hub's `published` repository**. See the
   [main collection README](https://github.com/bigpandaio/bigpanda-ansible#publishing-to-automation-hub)
   for upload steps.
3. A **BigPanda User API Key** with permission to act on incidents.
   Create one at *BigPanda → Settings → API Keys*.

### High-level steps

1. **Wire AAP to your Hub.** Generate an API token in Hub
   (*Automation Content → API Token → Load token*) and create a Galaxy
   credential in Controller (*Infrastructure → Credentials*,
   type **Ansible Galaxy/Automation Hub API Token**, URL
   `https://<your-aap-host>/api/galaxy/`).
   Then attach that credential to the **Default Organization** under
   *Access Management → Organizations → Default → Galaxy credentials*.
2. **Create a Project** in *Automation Execution → Projects* with:
   - Source control type: **Git**
   - URL: `https://github.com/bigpandaio/bigpanda-ansible-test.git`
   - Branch: `master`
   - **Update revision on launch:** ON (so changes to this repo are picked up
     automatically before every job).
3. **Create one Job Template per playbook** (see tables below). The Project
   sync also installs the `bigpanda.incident` collection automatically from
   the `collections/requirements.yml` file in this repo.

---

## Job Template configuration

### Common settings (every template)

| Field | Value |
|---|---|
| **Job type** | `Run` |
| **Inventory** | `Demo Inventory` *(or any inventory with localhost)* |
| **Project** | the Project you created above |
| **Execution environment** | default |
| **Credentials** | *(none — Surveys handle secrets)* |

### Recommended launch order

| Order | Template | Why this order |
|---|---|---|
| 1 ✅ | Comment | Cheapest test — proves auth + routing without changing incident state |
| 2 | Add Tag | Non-destructive metadata change |
| 3 | Assign User | Non-destructive — easy to undo |
| 4 | Resolve Alert | Resolves an alert (still inside incident) |
| 5 | Split | Splits alerts out into a new incident |
| 6 | Merge | Requires ≥2 disposable incidents |
| 7 | Resolve Incident 🛑 | **Last.** Closes the incident permanently |

### Survey-question specs per template

#### 1. `BigPanda - Comment` — playbook `playbooks/smoke_comment.yml`

| # | Question | Variable | Type | Required | Default | Description |
|---|---|---|---|---|---|---|
| 1 | BigPanda Environment ID | `environment_id` | Text | yes | — | UUID from the BigPanda environment URL |
| 2 | BigPanda Incident ID | `incident_id` | Text | yes | — | UUID from the incident URL |
| 3 | BigPanda API Token | `api_token` | **Password** | yes | — | User API Key |
| 4 | Comment text | `msg` | Text | no | `Hello from AAP via Job Template` | Appears on the incident timeline |

#### 2. `BigPanda - Add Tag` — playbook `playbooks/add_tag.yml`

| # | Question | Variable | Type | Required | Default | Description |
|---|---|---|---|---|---|---|
| 1 | BigPanda Environment ID | `environment_id` | Text | yes | — | UUID |
| 2 | BigPanda Incident ID | `incident_id` | Text | yes | — | UUID |
| 3 | BigPanda API Token | `api_token` | **Password** | yes | — | User API Key |
| 4 | Tag name (tag_id) | `tag_id` | Text | yes | — | Must already exist in your env, e.g. `priority`, `owner`, `service` |
| 5 | New tag value | `tag_value` | Text | yes | — | e.g. `P3`, `platform-team` |

#### 3. `BigPanda - Assign User` — playbook `playbooks/assign.yml`

| # | Question | Variable | Type | Required | Default | Description |
|---|---|---|---|---|---|---|
| 1 | BigPanda Environment ID | `environment_id` | Text | yes | — | UUID |
| 2 | BigPanda Incident ID | `incident_id` | Text | yes | — | UUID |
| 3 | BigPanda API Token | `api_token` | **Password** | yes | — | User API Key |
| 4 | Assignee user ID | `assignee_id` | Text | yes | — | BigPanda user UUID (SCIM `/Users` API, or BigPanda User Management) |

#### 4. `BigPanda - Resolve Alert` ⚠️ — playbook `playbooks/resolve_alert.yml`

| # | Question | Variable | Type | Required | Default | Description |
|---|---|---|---|---|---|---|
| 1 | BigPanda Environment ID | `environment_id` | Text | yes | — | UUID |
| 2 | BigPanda API Token | `api_token` | **Password** | yes | — | User API Key |
| 3 | Alert IDs (JSON array) | `alert_ids` | **Textarea** | yes | `["alert-uuid-1"]` | JSON list. Example: `["5d09d221aebaec1c43ccd448"]`. Max 500 per call |

#### 5. `BigPanda - Split Incident` ⚠️ — playbook `playbooks/split.yml`

| # | Question | Variable | Type | Required | Default | Description |
|---|---|---|---|---|---|---|
| 1 | BigPanda Environment ID | `environment_id` | Text | yes | — | UUID |
| 2 | Source incident ID | `incident_id` | Text | yes | — | The incident to split alerts OUT of |
| 3 | BigPanda API Token | `api_token` | **Password** | yes | — | User API Key |
| 4 | Alert IDs to split off (JSON array) | `alert_ids` | **Textarea** | yes | `[]` | Alerts to move to a new incident |
| 5 | Reason / comment | `comment` | Text | no | `Split via AAP smoke test` | Timeline note |

#### 6. `BigPanda - Merge Incidents` ⚠️ — playbook `playbooks/merge.yml`

| # | Question | Variable | Type | Required | Default | Description |
|---|---|---|---|---|---|---|
| 1 | BigPanda Environment ID | `environment_id` | Text | yes | — | UUID |
| 2 | Target incident ID | `incident_id` | Text | yes | — | The incident that absorbs the others |
| 3 | BigPanda API Token | `api_token` | **Password** | yes | — | User API Key |
| 4 | Source incident IDs (JSON array) | `source_incidents` | **Textarea** | yes | `[]` | Incidents to merge INTO the target |
| 5 | Reason / comment | `comment` | Text | no | `Merged via AAP smoke test` | Timeline note |

#### 7. `BigPanda - Resolve Incident` 🛑 — playbook `playbooks/resolve.yml`

| # | Question | Variable | Type | Required | Default | Description |
|---|---|---|---|---|---|---|
| 1 | BigPanda Environment ID | `environment_id` | Text | yes | — | UUID |
| 2 | BigPanda Incident ID | `incident_id` | Text | yes | — | **Will be closed** |
| 3 | BigPanda API Token | `api_token` | **Password** | yes | — | User API Key |
| 4 | Resolution comment | `resolution_comment` | Text | no | `Resolved via AAP smoke test` | Posted before close |

### AAP Survey-field mapping

When you build each Survey question in the AAP UI, the column names in the
tables above map to these AAP fields:

| Table column | AAP Survey field |
|---|---|
| Question | **Question name** |
| Variable | **Answer variable name** |
| Type | **Answer type** dropdown |
| Required | **Required** toggle |
| Default | **Default answer** |
| Description | **Description** (shown as help text) |

---

## Running locally (no AAP)

The same playbooks work from a laptop if you want a quick sanity check
before wiring up AAP:

```bash
# install ansible-core + the collection
brew install ansible
ansible-galaxy collection install bigpanda.incident

# run any playbook with --check first (no network calls)
ansible-playbook -i 'localhost,' -c local --check \
  -e environment_id=<env-uuid> \
  -e incident_id=<inc-uuid> \
  -e api_token=<your-token> \
  playbooks/smoke_comment.yml

# remove --check to make the real API call
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Project sync fails with `bigpanda.incident not found` | The EE that ran ansible-galaxy doesn't know about your local Hub. | Generate a Hub API token, create a `Ansible Galaxy/Automation Hub API Token` credential pointing at `https://<aap>/api/galaxy/`, and attach it under *Access Management → Organizations → Default → Galaxy credentials*. |
| Job fails with HTTP 401 | Bad or expired BigPanda API key. | Mint a fresh one in BigPanda → Settings → API Keys, update the Survey answer. |
| Job fails with HTTP 404 + `Tag ID … does not exist` | The `tag_id` you supplied isn't a tag in your BigPanda environment. | Open an incident in BigPanda and use one of the tag names you see in the right-hand panel. |
| `bigpanda-ansible-test` repo changes not picked up | Project not re-synced. | In the Project row, click the ↻ sync icon, or enable **Update revision on launch** on the Project. |

---

## License

[GPL-2.0-or-later](https://www.gnu.org/licenses/gpl-2.0.txt), matching the
upstream collection.