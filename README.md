# vks-addons

A Claude Code skill packaging hard-won, verified knowledge of the VKS (vSphere Kubernetes
Service) addon system: the `AddonConfigDefinition`, `Addon`, `AddonRelease`,
`AddonRepository`, `AddonInstall` and `AddonConfig` resources under
`addons.kubernetes.vmware.com` on a vSphere Supervisor.

It exists because none of this is in the docs and it took real probing against a live
Supervisor to establish. The skill loads automatically when a session works with VKS
addons, so the constraints, delivery mechanics, template dialect and live verification
probes do not have to be re-derived.

## What it covers

- The seven-kind resource model and how they relate.
- The hard constraints: the manager-only validating webhooks on `Addon` and
  `AddonRelease` (RBAC does not help), why `AddonConfigDefinition` is creatable but inert
  alone, and why an `AddonRepository` is the only route.
- Delivery mechanics: shipping a hand-written ACD in a Package annotation (gzip+base64),
  and the payload-as-data values-Secret pattern.
- The template dialect (Go `text/template` + sprig + `toYaml`, ytt in the guest) and the
  body-only, static-GVK output rule.
- Live verification probes: testing the webhook lock with a minted service-account token,
  pulling a guest kubeconfig, decoding a shipped ACD as a reference, testing the guest
  render with an inline package.
- A decision guide for imgpkg vs helm `AddonRepository` vs a Supervisor Service wrapper.

Verified against VKS 3.7 (Supervisor and guest). Where a fact is inferred rather than
tested, the skill says so.

## Install

As a Claude Code plugin, from GitHub:

```
/plugin marketplace add warroyo/vks-addons
/plugin install vks-addons@vks-addons
```

Or from a local clone:

```
/plugin marketplace add /path/to/vks-addons
/plugin install vks-addons@vks-addons
```

Then reload plugins (`/reload-plugins`) or restart. Invoke it directly with
`/vks-addons`, or let it load automatically when a task involves VKS addons.

## Layout

```
vks-addons/
├── .claude-plugin/
│   ├── plugin.json         # plugin manifest
│   └── marketplace.json    # marketplace manifest (this repo is its own marketplace)
└── skills/
    └── vks-addons/
        └── SKILL.md        # the skill
```

## License

MIT. See [LICENSE](LICENSE).
