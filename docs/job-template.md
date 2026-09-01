# Ansible Tower job template

The pipeline's `deployTo()` calls one job template for every environment. What
changes between environments is the `limit` (which hosts) and the credential —
both passed at launch, neither baked into the template.

## Configuration

| Field | Value |
| --- | --- |
| Name | `jt_app_deploy` |
| Job type | Run |
| Inventory | `app_hosts` |
| Project | the project pointing at this repository |
| Playbook | `ansible/deploy_image/1.0/deploy_image.yml` |
| Credentials | a machine credential for the target hosts |
| Verbosity | 1 (Verbose) |
| Privilege escalation | enabled — the playbook uses `become` |

## Prompt on launch

Enable **Limit**, **Credentials**, and **Variables**.

This is the step that is easy to miss and awkward to debug. Without these prompts,
Tower ignores the `limit`, `credential`, and `extraVars` the pipeline sends and runs
with the template's own defaults instead — so a deployment aimed at one environment
can quietly run somewhere else, or with the wrong image reference, and still report
success.

## Inventory

One group per environment:

```ini
[dev]
app-dev-01

[stage]
app-stage-01

[prod]
app-prod-01
app-prod-02
```

The pipeline passes individual hostnames as the `limit`, so groups are not strictly
required. They are worth having anyway: they document the estate, and they let the
pipeline target a whole environment (`limit: dev`) instead of a host list once the
inventory is authoritative.

## Credentials

One machine credential per environment is the usual arrangement, mapped in the
pipeline's `credentialForEnvironment()`. A single credential across all environments
is simpler but gives every deployment the reach of the most sensitive one.

## Checking it independently

Before wiring the pipeline, launch the template by hand against one host:

```
limit:      app-dev-01
extra vars: deploy_ref: registry.example.com/example/app:latest
            latest_ref: registry.example.com/example/app:latest
```

If that works, the playbook, inventory, and credentials are correct, and anything
that fails afterwards is on the pipeline side. Splitting the problem this way saves
a long afternoon.
