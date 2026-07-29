---
name: vks-addons
description: >-
  Build, install, or debug VKS (vSphere Kubernetes Service) addons: the
  AddonConfigDefinition / Addon / AddonRelease / AddonRepository / AddonInstall /
  AddonConfig resource family under addons.kubernetes.vmware.com on a vSphere Supervisor.
  Use when authoring an addon, packaging one as a Carvel package repository, wiring
  per-cluster config through an AddonConfig, deciding between imgpkg and helm
  AddonRepositories, or troubleshooting why addon resources will not create or reconcile.
  Covers the manager-only webhooks, the payload-as-data pattern, the template dialect,
  and live verification probes.
metadata:
  source: empirical, verified against VKS 3.7 (Supervisor + guest)
---

# Working with VKS addons

Hard-won, verified knowledge of the VKS addon system. Facts here were checked against a
live Supervisor and guest cluster (VKS 3.7). Where something is inferred rather than
tested, it says so. Trust shipped definitions over CRD field descriptions, which are
sometimes stale.

## The resource model

Seven kinds under `addons.kubernetes.vmware.com`, in two groups.

Manager-owned, created for you, live in `vmware-system-vks-public`:

- **AddonConfigDefinition (ACD)**: the addon's schema (`openAPIV3Schema`) plus
  `templateOutputResources` (what it renders per cluster) and optional
  `templateInputResources` (things it reads, like a Cluster CR or a ConfigMap).
- **Addon**: groups releases of one addon. Mostly metadata (`displayName`, `description`).
- **AddonRelease**: binds an `Addon` to an `AddonConfigDefinition` version, carries
  `kubernetesVersionConstraints`, and may carry `spec.package`. If it has `spec.package`,
  the guest gets a PackageInstall; if not, the addon is package-free.

Admin- or tenant-creatable, the levers you actually pull:

- **AddonRepository** + **AddonRepositoryInstall**: an admin creates these; the manager
  reconciles them and materialises the three kinds above. This is the only way to get
  them created (see constraints).
- **AddonInstall**: a tenant attaches an addon to clusters via a label selector.
  References the `Addon` by `spec.addonRef`. Created in the cluster's Supervisor namespace.
- **AddonConfig**: per-cluster values. Must be named `<cluster-name>-<addon-name>`, in the
  cluster's Supervisor namespace, with annotation
  `clusteraddon.addons.kubernetes.vmware.com/owned-for-deletion: "true"` so it is garbage
  collected with the ClusterAddon.

## The hard constraints (the expensive lessons)

**Addon and AddonRelease are manager-locked.** Validating webhooks
(`addon.validating.vmware.com`, `addonrelease.validating.vmware.com`) reject every
mutating operation on them from any client that is not the VKS addon manager's own
service account. RBAC does not help: a service account with `create` still gets denied at
admission, and create, update, client-side apply and server-side apply all fail
identically. The manager runs with `--enable-webhook-client-verification`; the check is on
caller identity. Errors look like:

```
Addon is a managed resource and CREATE is not allowed
operation on an AddonRelease is only allowed from VKS addon manager service account
```

**AddonConfigDefinition is NOT locked, but it is inert alone.** You can create an ACD
directly, but nothing consumes one that is not selected by an AddonRelease, and an
AddonInstall requires the Addon to reconcile. So authoring only the ACD gets you nothing.

**The only route to create the three kinds is an AddonRepository.** Two flavours:

- `fetch.imgpkgBundle`: a Carvel package repository. The manager reads the Package(s) and
  creates the addon CRs. The AddonRelease gets `spec.package` set, so the guest gets a
  PackageInstall. This preserves a hand-written ACD (it travels in a Package annotation).
- `fetch.helmRepository`: the manager generates the ACD from the chart, with a fixed
  `helmValues`/`helmOptions` schema, and the AddonRelease is package-free. You lose control
  of the ACD schema and pull in `helm-controller` as a dependency.

**The guest fetch is unavoidable with imgpkg.** `spec.package` forces a guest
PackageInstall, and the guest acquires the package through a repository that fetches a
bundle. There is no package-free-and-fetch-free path through the addon system. Guest pulls
route through an in-cluster proxy (`mgmt-image-proxy.kube-system.svc` / the depot) that the
Supervisor feeds, not straight to an external registry.

**Supervisor Service packages get their namespace rewritten.** A Supervisor Service is
deployed with kapp's namespace rewrite, so every namespaced resource it applies lands in
the service's own namespace (`svc-<name>-<id>`), not where the YAML says. Cluster-scoped
resources are exempt. The manager does reconcile an AddonRepository/AddonRepositoryInstall
created in a service namespace, so a thin Supervisor Service wrapping those two CRs works.

**Placement and required labels/annotations.**
- The three addon kinds must be in `vmware-system-vks-public`.
- Each must carry `addon.kubernetes.vmware.com/addon-name` = the addon name (webhook
  rejects without it). Shipped addons also carry `addon.kubernetes.vmware.com/addon-namespace`.
- An AddonRepository must carry `addons.kubernetes.vmware.com/package-offerings` (a JSON
  listing of packages/versions it offers) or the webhook rejects it.

## Delivery: the ACD-in-annotation and payload-as-data patterns

**A hand-written ACD ships inside a Package.** Put the full AddonConfigDefinition YAML into
the Package's `addons.kubernetes.vmware.com/addon-config-definition` annotation as
**gzip+base64**. The manager decodes it and creates the ACD. This is how cilium and every
package-derived addon delivers its definition, and it is how you keep your own schema
through the AddonRepository route. Encode with `gzip -c | base64 -w0`.

**Per-cluster payload flows as data through a values Secret.** The standard pattern (see
cilium):

1. Tenant writes values in `AddonConfig.spec.values`.
2. The ACD's `templateOutputResources` render a **Secret** carrying those as data values.
   Use a `supervisorNamespaceOutput` Secret with `referenceType: ValuesRef` (wired into the
   guest PackageInstall's values) plus a `targetClusterOutput` Secret of the same name in
   the guest package namespace (`vmware-system-tkg`). Body shape:
   `stringData: {values.yaml: |\n  <data values>}` and `type: Opaque`.
3. The Package's own render (ytt, in the guest) reads those values and emits resources.

To pass arbitrary YAML through ytt data values safely (ytt interprets `#@`): JSON-encode
it in the ACD and decode in the package. E.g. structured resources as
`{{ .Values.resources | toJson | toJson }}` -> a JSON string; the package does
`json.decode`. Raw multi-doc YAML as `{{ $raw | toJson }}`; the package splits on the doc
separator and `yaml.decode`s each.

## Output templates are the resource body only

Verbatim from the CRD: a `template` is the resource specification excluding TypeMeta and
ObjectMeta. Consequences:
- No `apiVersion`, `kind`, or `metadata` in a template.
- One resource per output entry; no multi-document output.
- The GVK is a static literal on `targetClusterOutput`/`supervisorNamespaceOutput`. `name`
  and `namespace` are templatable, the GVK is not. This is why an addon cannot emit
  arbitrary payload kinds directly; render them in a package instead.

## The template dialect

`templateOutputResources[].template` is **Go `text/template` + sprig**, evaluated per
cluster by the addon controller (not CEL, despite a stale CRD field description). Plus the
Helm-style `toYaml`/`fromYaml`, which the controller registers although they are not sprig.
Context roots:

- `.Values.<field>` (capital V): the AddonConfig values, defaulted from the ACD schema.
- `.Dependencies.<inputName>`: a resolved `templateInputResource` (the whole object).
- `.Cluster` (e.g. `.Cluster.name`) and `.Addon`.

The Package's own guest-side render is **ytt** (`json`, `yaml`, `data` modules useful).

## Building an imgpkg addon-repository bundle

A Carvel package repository bundle:

```
.imgpkg/images.yml            # ImagesLock; empty (images: []) if the package is config-only
packages/<pkg-ref>/
  metadata.yml                # PackageMetadata
  <version>.yml               # Package, with the ACD in its annotation
```

The Package's `spec.template.spec` is `fetch` (use `inline` to avoid a second guest fetch
for the package content) / `template` (ytt) / `deploy` (kapp). Push with
`imgpkg push -b <registry>/<repo>:<version> -f <bundle-dir>`. Then an admin applies an
AddonRepository pointing at that image plus an AddonRepositoryInstall.

## Live verification probes

You can learn a lot without a full build/upload cycle.

- **Test the webhook lock** without publishing anything: mint a token for a service account
  that has RBAC on the kind, and attempt the create.
  ```sh
  kubectl -n <ns> create token <sa> --duration=10m   # build a kubeconfig with it
  # then, as that token, kubectl create the Addon/AddonRelease -> observe admission denial
  ```
  If it succeeds, the webhook policy has been relaxed.
- **Reach a guest cluster** from the Supervisor: pull its kubeconfig.
  ```sh
  kubectl -n <cluster-ns> get secret <cluster>-kubeconfig -o jsonpath='{.data.value}' | base64 -d > guest.kubeconfig
  ```
- **Decode a shipped ACD** to use as a reference implementation:
  ```sh
  kubectl -n vmware-system-vks-public get package <pkg> \
    -o jsonpath='{.metadata.annotations.addons\.kubernetes\.vmware\.com/addon-config-definition}' \
    | base64 -d | gunzip
  ```
- **Test the guest render** directly: create an inline-fetch kapp-controller `App`, or a
  `Package` + `PackageInstall` with inline content and a values Secret, in a scratch guest
  namespace. Confirms the ytt render and that inline fetch pulls no image. kapp-controller
  is in `tkg-system`; carvel CRDs and guest PackageRepositories are present.
- **Read reconcile errors**: `AddonRepositoryInstall`, `PackageInstall` and `ClusterAddon`
  all expose `status.usefulErrorMessage` / conditions. The addon controller reports template
  errors in the ClusterAddon conditions, per cluster, after install.
- **Check who reconciles what**: an AddonInstall creates a guest PackageInstall (annotation
  `addons.kubernetes.vmware.com/addoninstall-name`). Guest Package CRs carry
  `packaging.carvel.dev/package-repository-ref` if fetched by a repo, or none if manager-synced.

## Decision guide

- Want your own tenant-facing schema and to run on kapp-controller only: **imgpkg
  AddonRepository** with the ACD in the Package annotation. Accept the guest package fetch.
- Fine with a helm-values interface and want a package-free AddonRelease: **helm
  AddonRepository**. Accept the generated ACD and helm-controller dependency.
- Want single-upload delivery through the vCenter Services catalog: wrap the two
  AddonRepository CRs in a **Supervisor Service** (they land in the service namespace; the
  manager still reconciles them). Otherwise just `kubectl apply` the two CRs.
- Tempted to create the Addon/AddonRelease yourself (Job, controller, apply): **do not**,
  it is webhook-blocked by caller identity. Only the manager can, only via AddonRepository.

## Reference implementation

`warroyo/dayzero-addon-service` is a worked example: a generic "apply arbitrary YAML at
provisioning" addon built as an imgpkg AddonRepository with a hand-written ACD and a native
render package, plus a Supervisor Service wrapper and both install methods. Its
`docs/design.md` records the package-free design (see *Package-free addons are legal* and
*Alternatives rejected*), and `docs/verify.md` records the probes it was checked with.
