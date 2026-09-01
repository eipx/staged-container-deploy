# Staged Container Deploy

A Jenkins + Ansible pattern for deploying a container image across environments
in a controlled, auditable, staged sequence.

One pipeline run can build an image, push it, and roll it out to DEV, then STAGE,
then PROD, pausing for human approval between environments. The same pipeline can
also deploy an *existing* image — by build number, commit hash, or digest — without
rebuilding anything.

## Why this exists

Teams that build images in CI often deploy them by hand: `docker pull` on each host,
one environment at a time, whenever someone remembers. That drifts — environments end
up running different images, and nobody can say which. This pattern closes the gap
without introducing an orchestrator.

It suits estates where:

- containers run directly on hosts (systemd/docker), not on Kubernetes
- deployment means *refresh the image*, not *restart a long-lived service*
- promotion between environments needs a human in the loop
- deployments must be auditable after the fact

## How it works

```
Jenkins pipeline
  ├── Build           build the image, tag it (build number + short commit hash)
  ├── Push            push all tags to the registry
  ├── Resolve         decide which image reference this run deploys
  ├── Deploy DEV      → Ansible Tower job template → playbook → hosts
  ├── [approval gate]
  ├── Deploy STAGE    → same template, different hosts
  ├── [approval gate]
  └── Deploy PROD     → same template, different hosts
```

The playbook on each host does four things: pull the requested image, retag it as
`latest` so consumers pick it up, prune dangling images, and write a digest-stamped
line to the system log.

## Modes

The pipeline takes a `MODE` parameter with three values:

| Mode | Build | Deploy | Use for |
| --- | --- | --- | --- |
| `Build only` | yes | no | CI on merge — the safe default for automatic triggers |
| `Build and deploy` | yes | yes | one-click build and staged rollout |
| `Deploy only` | no | yes | promote an existing image, or roll back to a previous one |

`Build only` is the default deliberately: pipelines triggered by SCM changes or by an
upstream job run with parameter defaults, so an automatic trigger can never deploy.

## Image references

Every build produces three names for the same bytes:

| Reference | Answers | Example |
| --- | --- | --- |
| build number | which CI run produced it | `registry.example.com/example/app:142` |
| short commit hash | which source built it | `registry.example.com/example/app:a1b2c3d4` |
| digest | which exact bytes | `registry.example.com/example/app@sha256:...` |

`Deploy only` accepts any of the three. The build also stamps the full commit into the
image as an OCI label, so any host can answer "what source is running here?":

```bash
docker inspect --format '{{index .Config.Labels "org.opencontainers.image.revision"}}' \
  registry.example.com/example/app:latest
```

## Audit trail

Container tooling records when an image was *built*, never when it was *deployed*.
The playbook closes that gap by writing to the host's system log on every deploy:

```bash
journalctl -t app-deploy
```

Each entry carries the deployed reference, its digest, and the CI build URL, so a host
can be asked what it is running and where that came from — long after the CI build
records have rotated away.

## Layout

```
Jenkinsfile                              Ansible Tower transport (recommended)
Jenkinsfile.ssh                          direct SSH transport (no Tower dependency)
ansible/deploy_image/1.0/deploy_image.yml   the deployment playbook
docs/job-template.md                     Ansible Tower job template configuration
```

## Two transports

The deployment step is isolated in a single `deployTo()` function so the transport is
swappable without touching stages, parameters, or gates.

**Ansible Tower** (`Jenkinsfile`) — the pipeline calls a job template, which runs the
playbook against the target host. Credentials are centrally managed, and Tower keeps
its own durable job history. Use this where Tower exists, especially if CI agents
cannot reach every environment directly.

**Direct SSH** (`Jenkinsfile.ssh`) — the pipeline SSHes to each host and runs the same
commands inline. No Tower dependency, useful as a starting point or where Tower is not
available. The trade-off is that the deploy key lives in the CI credential store and
deployment history lives only in build logs, which rotate.

## One registry, or several

The pipeline assumes every environment can pull from the same registry. That is the
common case, and it keeps the model honest: build once, push once, and every host pulls
the identical artifact.

Segmented estates often break that assumption — environments sit in different network
zones, each with its own registry, and a host in one zone has no route to another zone's
registry. Every deployment involves two hops, and they fail independently:

| Hop | Fails when |
| --- | --- |
| CI agent → host | the agent cannot reach that environment's network |
| host → registry | that environment has no route to the registry holding the image |

A job template fixes the first hop, not the second. If a deployment reports that it cannot
pull, check the second hop before assuming the transport is at fault — the symptom of a
missing route looks nothing like a permissions problem, and the playbook will have run
perfectly up to that point.

Where environments have separate registries, *promote* the image rather than rebuilding
it — a rebuild produces different bytes and defeats the point of deploying a known
artifact:

```groovy
stage( 'Promote to STAGE registry' )
{
    when { expression { params.MODE != 'Build only' } }
    steps
    {
        sh "skopeo copy docker://${env.DEPLOY_REF} " +
           "docker://stage-registry.example.com/example/app:${env.BUILD_ID}"
    }
}
```

Then make the registry per-environment the way credentials already are, and hand the
promoted reference to `deployTo()`. Verify by digest: a copy is byte-identical, so both
registries report the same `sha256:`, which is the evidence that nothing changed as the
image moved between zones.

## Setup

1. Push the playbook to a repository your Tower instance can read.
2. Create the job template described in `docs/job-template.md`.
3. Create an inventory with a group per environment.
4. Adjust the `environment` block in the `Jenkinsfile` — registry, namespace, image name.
5. Set host defaults in the pipeline parameters, or pass them per run.

Both pipelines assume the image build itself needs registry credentials on the build
agent; provide those the way your CI does today.

## Notes

- `prune -f` removes dangling images only. Tagged images accumulate; if that matters on
  your hosts, extend the playbook with an explicit retention task rather than widening
  the prune.
- Deploying an image a host already has is a no-op, and the playbook reports it as such.
  The audit log entry is still written.
- Rolling back is a `Deploy only` run against a previous reference. It is deliberately a
  human decision, not an automatic reaction to a failed deploy.

## License

MIT
