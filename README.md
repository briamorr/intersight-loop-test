# NTP Policy Tagging Playbooks

## Architecture

This repository now uses a 3-playbook model:

1. `set_ntp_policy_tags_multi.yml`
- Unified multi-policy wrapper.
- Supports both operations:
  - `tag_operation: set`
  - `tag_operation: cleanup`
- Input is `policy_tags_yaml` (YAML list from AAP survey or CLI).
- Runs one worker per policy and processes workers in parallel.

2. `set_ntp_policy_tags.yml`
- Single-policy tag set/update playbook.
- Merges requested tags with existing tags.

3. `cleanup_ntp_policy_tags.yml`
- Single-policy tag cleanup playbook.
- Removes existing tags by matching requested `Key` values.

## AAP Usage (Recommended)

Use `set_ntp_policy_tags_multi.yml` in AAP job/workflow templates for both set and cleanup.

### Required variables

- `policy_tags_yaml` (survey textarea)
- `tag_operation` (`set` or `cleanup`)
- `api_key_id` and `api_private_key` (from AAP credential injection)

### Survey example for set

```yaml
- policy_name: NTP-Production
  tags:
    - Key: Environment
      Value: Production
    - Key: Owner
      Value: NetOps
- policy_name: NTP-Lab
  tags:
    - Key: Environment
      Value: Lab
```

Run with:

```yaml
tag_operation: set
```

### Survey example for cleanup

```yaml
- policy_name: NTP-Production
  tags:
    - Key: Environment
    - Key: Owner
- policy_name: NTP-Lab
  tags:
    - Key: Environment
```

Run with:

```yaml
tag_operation: cleanup
```

## CLI Examples

### Multi-policy set

```bash
ansible-playbook -i inventory set_ntp_policy_tags_multi.yml \
  -e "tag_operation=set" \
  -e "api_key_id=${INTERSIGHT_API_KEY_ID}" \
  -e "api_private_key=${INTERSIGHT_API_PRIVATE_KEY}" \
  -e 'policy_tags_yaml=[{"policy_name":"NTP-Prod","tags":[{"Key":"Env","Value":"Prod"}]}]'
```

### Multi-policy cleanup

```bash
ansible-playbook -i inventory set_ntp_policy_tags_multi.yml \
  -e "tag_operation=cleanup" \
  -e "api_key_id=${INTERSIGHT_API_KEY_ID}" \
  -e "api_private_key=${INTERSIGHT_API_PRIVATE_KEY}" \
  -e 'policy_tags_yaml=[{"policy_name":"NTP-Prod","tags":[{"Key":"Env"}]}]'
```

### Single-policy set

```bash
ansible-playbook -i inventory set_ntp_policy_tags.yml \
  -e "ntp_policy_name=NTP-Prod" \
  -e 'policy_tags=[{"Key":"Env","Value":"Prod"}]' \
  -e "api_key_id=${INTERSIGHT_API_KEY_ID}" \
  -e "api_private_key=${INTERSIGHT_API_PRIVATE_KEY}"
```

### Single-policy cleanup

```bash
ansible-playbook -i inventory cleanup_ntp_policy_tags.yml \
  -e "ntp_policy_name=NTP-Prod" \
  -e 'policy_tags=[{"Key":"Env"}]' \
  -e "api_key_id=${INTERSIGHT_API_KEY_ID}" \
  -e "api_private_key=${INTERSIGHT_API_PRIVATE_KEY}"
```

## Behavior Summary

- `set` operation:
  - Preserves existing tags not in the incoming key set.
  - Overwrites values for matching keys.
  - Adds new keys.

- `cleanup` operation:
  - Removes existing tags whose `Key` is listed in input.
  - Leaves all other tags unchanged.

- Multi wrapper (`set_ntp_policy_tags_multi.yml`):
  - Continues processing other policies if one fails.
  - Shows success/failure summary.
  - Fails overall run if any policy failed.

## Notes

- `cleanup_ntp_policy_tags_multi.yml` no longer exists after refactor.
- Use `set_ntp_policy_tags_multi.yml` + `tag_operation=cleanup` for multi-policy cleanup.
