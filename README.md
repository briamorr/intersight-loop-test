# NTP Policy Tagging Playbooks

## Architecture

Two playbooks working together for flexible tag management:

### 1. `set_ntp_policy_tags.yml` (Single Policy)
Core playbook that applies tags to **one** NTP policy. 

**Standalone usage (CLI):**
```bash
ansible-playbook -i inventory set_ntp_policy_tags.yml \
  -e "ntp_policy_name=MyPolicy" \
  -e 'policy_tags=[{"Key":"Env","Value":"Prod"},{"Key":"Owner","Value":"NetOps"}]'
```

**Expected variables:**
- `ntp_policy_name`: Exact name of the policy
- `policy_tags`: List of tag dictionaries with `Key` and `Value`

**What it does:**
1. Looks up policy Moid by name
2. Reads existing tags on the policy
3. Merges supplied tags with existing ones (supplied keys override, new keys are added)
4. PATCHes the policy with merged tags

---

### 2. `set_ntp_policy_tags_multi.yml` (Multiple Policies - AAP Wrapper)
Wrapper that handles **multiple policies** with **different tags per policy**.

**AAP Job Template setup:**
Create a survey with this field:
- **Label:** Policy Tags (YAML)
- **Variable:** `policy_tags_yaml`
- **Type:** Textarea
- **Help text:** Enter YAML with policy_name and tags list

**Example survey input:**
```yaml
- policy_name: NTP-Production
  tags:
    - Key: Environment
      Value: Production
    - Key: CostCenter
      Value: IT-001
    - Key: Owner
      Value: netops-team
- policy_name: NTP-Staging
  tags:
    - Key: Environment
      Value: Staging
    - Key: CostCenter
      Value: IT-002
- policy_name: NTP-Lab
  tags:
    - Key: Environment
      Value: Lab
```

**CLI usage (if needed):**
```bash
ansible-playbook -i inventory set_ntp_policy_tags_multi.yml <<EOF
policy_tags_yaml: |
  - policy_name: NTP-Production
    tags:
      - Key: Env
        Value: Prod
  - policy_name: NTP-Lab
    tags:
      - Key: Env
        Value: Lab
EOF
```

**What it does:**
1. Parses the YAML input
2. Validates structure (ensures each entry has policy_name and tags)
3. Loops through each policy/tag combination
4. For each, includes `include_single_policy_task.yml` to apply tags
5. Tracks successes/failures
6. Displays summary with pass/fail counts

---

### 3. `include_single_policy_task.yml` (Reusable Tasks)
Helper file containing the core tagging logic, suitable for inclusion in loops.

**Called by:** `set_ntp_policy_tags_multi.yml` (via `include_tasks` in a loop)

---

## AAP Job Template Configuration

**Job Template Name:** `Tag NTP Policies`

| Setting | Value |
|---------|-------|
| Playbook | `set_ntp_policy_tags_multi.yml` |
| Inventory | (your Intersight inventory) |
| Credentials | Custom Credential (Intersight API Key) |
| Enable privilege escalation | Off |
| Prompt on launch | ✓ Survey enabled |

**Survey Configuration:**
1. **Question 1: Policy Tags (YAML)**
   - Variable: `policy_tags_yaml`
   - Type: Textarea
   - Required: Yes
   - Help text: Paste YAML with policy definitions

**Credentials:**
The Custom Credential type should inject:
- `INTERSIGHT_API_KEY_ID` (env var)
- `INTERSIGHT_API_PRIVATE_KEY` (env var, path to PEM file)

---

## Tag Merging Logic

Both playbooks merge tags intelligently:

**Example:**
```
Policy currently has:  [{"Key":"Owner","Value":"Old","Key":"Retention","Value":"30days"}]
You supply:            [{"Key":"Owner","Value":"New"},{"Key":"CostCenter","Value":"IT-001"}]
Result:               [{"Key":"Retention","Value":"30days"},{"Key":"Owner","Value":"New"},{"Key":"CostCenter","Value":"IT-001"}]
```

- Existing tags with Keys **not in** your input are preserved
- Supplied Keys **override** existing values
- New Keys are **added**

---

## Error Handling

- **Single playbook** (`set_ntp_policy_tags.yml`): Fails immediately if policy not found or tagging fails
- **Multi playbook** (`set_ntp_policy_tags_multi.yml`): Continues processing all policies even if one fails, then displays summary and fails the job if any errors occurred

---

## Authentication

Both playbooks read from environment variables (set via AAP Credential):
```bash
export INTERSIGHT_API_KEY_ID="your-api-key-id"
export INTERSIGHT_API_PRIVATE_KEY="/path/to/intersight-key.pem"
```
