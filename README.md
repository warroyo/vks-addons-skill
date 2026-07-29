# vks-addons

A Claude Code skill covering the VKS (vSphere Kubernetes Service) addon system: the
`AddonConfigDefinition`, `Addon`, `AddonRelease`, `AddonRepository`, `AddonInstall` and
`AddonConfig` resources under `addons.kubernetes.vmware.com` on a vSphere Supervisor.

## Where this came from

All of it came out of building
[warroyo/dayzero-addon-service](https://github.com/warroyo/dayzero-addon-service), an addon
that applies arbitrary YAML at cluster provisioning time. Getting that to work meant
poking at a live VKS 3.7 Supervisor and guest cluster: reading the shipped CRDs and webhook
configurations, decoding an already shipped `AddonConfigDefinition`, minting a
service-account token to find out what the webhooks will actually admit, and rendering
templates in the guest to see what the controller does with them.

Those findings live in that repo's [`docs/design.md`](https://github.com/warroyo/dayzero-addon-service/blob/main/docs/design.md)
and [`docs/verify.md`](https://github.com/warroyo/dayzero-addon-service/blob/main/docs/verify.md).
This skill is the condensed version, so a session working on VKS addons starts with the
constraints, delivery mechanics, template dialect and verification probes instead of
working them out again.

## What it covers

- The seven kinds and how they relate.
- Why the validating webhooks on `Addon` and `AddonRelease` only admit the manager (RBAC
  does not help you here), why an `AddonConfigDefinition` on its own does nothing, and why
  an `AddonRepository` is the only way in.
- How to ship a hand-written ACD in a Package annotation (gzip+base64), plus the
  payload-as-data values-Secret pattern.
- The template dialect: Go `text/template` with sprig and `toYaml` on the Supervisor, ytt
  in the guest, and output that must be body-only with a static GVK.
- Probes for checking any of this against a running cluster. Minting a service-account
  token to test the webhook lock, pulling a guest kubeconfig, decoding a shipped ACD to use
  as a reference, rendering an inline package in the guest.
- When to reach for an imgpkg `AddonRepository`, a helm one, or a Supervisor Service
  wrapper.

It was all verified against VKS 3.7, Supervisor and guest. Where a fact is inferred rather
than tested, the skill says so.

## Install

As a Claude Code plugin, from GitHub:

```
/plugin marketplace add warroyo/vks-addons-skill
/plugin install vks-addons@vks-addons
```

Or from a local clone:

```
/plugin marketplace add /path/to/vks-addons-skill
/plugin install vks-addons@vks-addons
```

Then reload plugins (`/reload-plugins`) or restart. Invoke it directly with
`/vks-addons`, or let it load automatically when a task involves VKS addons.

## Layout

```
vks-addons-skill/
├── .claude-plugin/
│   ├── plugin.json         # plugin manifest
│   └── marketplace.json    # marketplace manifest (this repo is its own marketplace)
└── skills/
    └── vks-addons/
        └── SKILL.md        # the skill
```

## License

MIT. See [LICENSE](LICENSE).
