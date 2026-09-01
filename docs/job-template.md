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

Enable **Limit** and **Variables**.

This is the step that is easy to miss and awkward to debug. Without these prompts,
Tower ignores the `limit` and `extraVars` the pipeline sends and runs with the
template's own defaults instead — so a deployment aimed at one host can quietly run
somewhere else, or with the wrong image reference, and still report success.

Leave **Credentials** unprompted. The machine credential belongs on the template, and
a fixed credential with no prompt means a caller cannot swap it. The prompt is only
needed in the one setup that requires per-launch credential selection — a single
template serving environments with *different* machine credentials — in which case the
pipeline passes the credential name and this prompt is what allows Tower to accept it.

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

Two credentials exist in this arrangement, and they authenticate different hops:

- **Tower's machine credential** — how Tower reaches the target hosts. Attached to
  the job template, stored in Tower, never referenced by the pipeline.
- **The CI-to-Tower credential** — how the pipeline calls Tower's API to launch the
  template. Stored in the CI credential store. Use a service account: a pipeline
  authenticated as a person stops working when that person's password rotates, and
  every deployment is attributed to them regardless of who triggered it.

If environments need *different* machine credentials (separate zones, separate
accounts), create one credential per environment in Tower, enable the Credentials
prompt, and pass the name from the pipeline per call. Don't build that until an
environment actually requires it.

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
