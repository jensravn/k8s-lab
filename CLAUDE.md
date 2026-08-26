# k8s-lab

A personal lab for learning Kubernetes alongside the CKA course. Cheap on
purpose: the cluster is expected to be stopped, deleted and rebuilt often.

## Language

Everything in this repository is English — documentation, comments, commit
messages, identifiers.

## The README is the runbook

`README.md` is authoritative. It must be possible to delete the cluster
entirely and rebuild it from nothing using only this repository. Any change
that affects how the cluster is built belongs in the README in the same
commit as the change itself.

Two rules follow from that:

- **End state, not history.** The README describes what exists, never the
  migration that produced it. Someone building from scratch never had the
  thing that was replaced, so mentioning it only misleads.
- **No environment-specific values.** Capture IPs, resource IDs and node
  names into shell variables where they are created, and reference the
  variable afterwards. A rebuilt cluster gets different ones, and a command
  with a stale address in it fails in ways that look like something else.
- **Bindingly ordered.** Sections are a sequence, not a menu.

## Traffic

Gateway API — `GatewayClass`, `Gateway`, `HTTPRoute` — served by Envoy
Gateway. Not `Ingress`: the ingress-nginx project was retired in March 2026
and gets no further releases, bug fixes or security patches.

TLS certificates come from Let's Encrypt via cert-manager, which renews them
on its own.

## Images

The nodes are amd64. Build `--platform linux/amd64`; get it wrong and the pod
fails with `exec format error`.

Tag with a version or the commit SHA, never `:latest`. Kubernetes cannot tell
that a tag it already holds now points somewhere else, so a rollout does
nothing at all.

The GHCR package is private and stays private. Pods pull it through the
`ghcr` image pull secret, which is namespaced and has to live alongside them.

## Manifests

Applied with `kubectl apply -f manifests/<dir>/`.

Comments explain _why_ a field is set the way it is, or what breaks without
it. The field name already says what it does; a comment repeating it is
noise.

## Commits

Conventional Commits: `type(scope): subject`.
