Hello Fermyon Live Code Tuesday Audience!

# Flux SpinKube Demo

This repo shows how to run SpinKube on a Kubernetes cluster, with Flux 2.3!

## Based on

The structure of this repository is modeled after:

<https://github.com/fluxcd/flux2-kustomize-helm-example>

which shows more detail how to use Flux to order dependencies including Helm.

## How to use

To try this example on your own cluster, you'll fork this repo and bootstrap Flux.

#### Cluster Requirements

Currently you'll need to choose one of these options (based on the SpinKube quickstart):

* Create a Kubernetes cluster with a k3d image that includes the
  containerd-shim-spin prerequisite already installed
* Use kwasm-operator (or runtime-class-manager, in the future) - pick this
  option if your cluster has nodes, and you can access them
* Install a system extension - eg.
  [container-runtime/spin](https://github.com/siderolabs/extensions/tree/main/container-runtime/spin)
  for immutable host OS like Talos, who cannot use kwasm-operator

If you have physical access to your cluster nodes and are cluster-admin, just
resume all of these `Kustomizations` and `HelmReleases` in order.

Infrastructure goes before Apps. Then, the `HelmReleases` and some Flux
`Kustomizations` will be numbered for the demo, to see better what's happening.

You can just unpause everything and let Flux figure out the dependency order,
because we use `spec.dependsOn` in each resource if it has a dependency.

If you chose one of the options that pre-installs the containerd shim, then you
can skip "kwasm-operator" and proceed otherwise identically!

Watch the video if you have the patience for such things, or skip to the
[tl;dw](#tldw) if you'd like to immediately try it for yourself.

### Todo:

- [X] Add flux-system-webhook `Ingress` & Flux `Receiver`
- [X] Add `IngressRoutes` (external)
- [X] Add `flux-docs` `Ingress`
- [X] Add `simple-spinapp` `Ingress`
- [X] Patch/annotate the nodes using the `flux/ssa: merge` and `prune: false`
- [X] Fix numbers to include HelmReleases (cert-manager before configs)
- [ ] Test everything one more time (eg. `flux-oci`)
- [ ] Reset `flux-system` to prepare for bootstrap

### WARNING

Not for production, this is an example only! We provisioned a test cluster in
our home lab with Kamaji and KubeVirt using Cozystack, just for this demo!

If you are using a dev cluster like `vcluster` where your physical nodes are
borrowed somehow from a parent cluster, it will not work. Or if you are not
able to gain write access to the cluster nodes' host system in a privileged
way, and shim-spin is not installed on the hosts, then you may be out of luck.

We need privileged access to nodes to configure containerd to run `SpinApp`
Spin apps. In production, you will probably ask your platform team or cloud
vendor to provide this unless you are building it yourself. (See above, this
demo is not for production. Operationalize this knowledge at your own risk!)

The Kamaji cluster is provisioned with an Ingress, and we will be exposing it
to the public internet, but for your local trial if you are using k3d, you can
[set up your own Ingress](https://k3d.io/v5.3.0/usage/exposing_services/) and
use `/etc/hosts`, which is the easy way for just one single workstation.

Or even just use `kubectl port-forward` - since it is a demo, I guess it
doesn't need to be exposed to the public internet, does it? (Mine does...)

### The YouTube Video

SpinKube and Flux automated, on Kubernetes 1.30 w/ Kingdon Barrett

<https://www.youtube.com/watch?v=IXi2lhRjOlU>

I will be following these directions live, and then cleaning up after.

(If you are eager to try, you can already fork and follow along while the show
is live, but do take care to delete my `flux-system` from your fork before you
bootstrap! Or, bootstrap into your own directory, and copy from `cozy-test`.)

### tl;dw

1. Fork this repo

2. Bootstrap Flux in `clusters/cozy-test`, substituting your own user and repo fork name:

```
$ GITHUB_USER=kingdonb
$ GITHUB_REPO=spin-flux-tuesday-demo
$ flux bootstrap github --owner=$GITHUB_USER --personal --repository=$GITHUB_REPO --branch=main --path=clusters/cozy-test
```

3. We should see some `HelmReleases` if we check the Flux CLI, (and hopefully
   within a few moments the Capacitor Web UI has become ready too)

4. Step through the `HelmReleases`, they are all marked `suspend: true` in Git.
   Push a commit that un-suspends them all at once, or use the UI to do it.

5. Note the example SpinApp has come online ( 🤞😵 ) if you have your own spin
   app, go ahead and deploy it now.

6. Use `kubectl port-forward` to check the response from your spin app!

7. Find
   <https://www.spinkube.dev/docs/spin-plugin-kube/tutorials/autoscaler-support/>
   to try something a bit more difficult,<br/>or for the gentle intro check out
   <https://www.spinkube.dev/docs/spin-plugin-kube/>
   <br/>– if you're entirely new to the idea of running Spin apps on Kubernetes!

#### Troubleshooting

If your `SpinApp` is stuck in `ContainerCreating`, and you are using KWasm,
then you may simply need to [annotate the nodes](https://github.com/KWasm/kwasm-operator?tab=readme-ov-file#kwasm-operator).

```bash
for i in `kubectl get node -o name`; do kubectl annotate $i kwasm.sh/kwasm-node=true; done
```

If you get this error:

```
✔ component manifests are up to date
► installing components in "flux-system" namespace
✗ Kustomization/flux-system/flux-system dry-run failed: no matches for kind "Kustomization" in version "kustomize.toolkit.fluxcd.io/v1"
```

...Then either I forgot to clean up after my experiments in `flux-system`, or
the livestream hasn't finished yet. You can delete that directory from inside
your fork (`clusters/cozy-test/flux-system`) and just bootstrap Flux again.

## What is in the demo

Bootstrapping Flux onto a cluster is intended as a canonical way to seed the
cluster with a single source of truth.

When `flux bootstrap` creates the first `flux-system` `Kustomization` and
`GitRepository`, it should be clear this is for the Flux system only from the
klaxon warnings at the top of each file, `DO NOT EDIT`.

You can put anything you want in that directory, (but don't come cryin'...)

You are meant to create `Namespaces` and/or Tenants from `flux-system`,
depending on your requirements. We handled our tenant strategy through the
multi-cluster strategy.

So there are no tenants, only namespaces. All of our Flux resources can go in
the `flux-system` namespace grouped together for simplicity. We only need two,
"apps" and "infrastructure" so we may add any prerequisites early enough.

### Cluster

The cluster has only the bare minimum services to provide CNI, CSI, cluster
DNS, and `konnectivity` (which is a clue that we are not alone – there are
other clusters.)

Your cluster may differ from ours, it should be an empty cluster that you can
use for testing safely.

Our "cozy-test" Kubernetes and its control plane is provisioned by Cozystack.

#### Infrastructure

Some prerequisites are applied first, in the `infrastructure` Flux `Kustomization`.
Then, we can see `apps` come online without error as they wait for prerequisites.

##### Cert-Manager

We have a `cert-manager` `HelmRelease` copied from upstream.  It is configured
for automatic upgrades, and will readily install whatever the latest release is
in the `1.x` MAJOR series.

Thankfully, that means we can get the latest version with no intervention!

When we use this pattern in other places, we will continue to call it out.

<https://github.com/fluxcd/flux2-kustomize-helm-example?tab=readme-ov-file#infrastructure>

The example keeps dependencies simple and avoids explicit dependency between
`HelmReleases` because we really don't need to build a chain of dependencies.

`HelmReleases` that go in the `infrastructure` Flux `Kustomization` are applied
first. The `apps` Flux `Kustomization` won't be applied before `infrastructure`
is Ready! This way we won't see any errors, 🤞.

What else is in `infrastructure`? ... keep reading, and let's find out!

##### Runtime Class

We have copied the latest version of the `spin-operator.runtime-class.yaml`
from Spin Operator `v0.2.0`, (you may wish to update it if this has become
stale.)

This simply alerts the cluster that it can schedule any pods with a
`runtimeClassName` set that matches `spin`.

(We don't need to know this, as it is a detail that SpinKube has intentionally
abstracted away from users!)

##### Custom Resource Definitions

A CRD called `SpinApp` and a few others, like I said, users only need to know
about `SpinApp` as long as they have competent platform teams to set this up
for them. We've also got one called `SpinAppExecutor` that will come up later.

Sometimes Helm charts will manage the CRD, but something tells me the authors
of this operator are very familiar with the issues surrounding CRDs and Helm!

Here's a bit of Flux documentation to support [Installing and Upgrading CRDs
with `HelmReleases`](https://fluxcd.io/flux/components/helm/helmreleases/#controlling-the-lifecycle-of-custom-resource-definitions)
and the issues surrounding that.

More information that users do not need to know, (usually don't, \*unless they
are both Helm users and CRD authors, which we might try later...)

This information has been used above when we installed `cert-manager` but for
SpinKube, since we are following the Quickstart, we don't need it.

##### Spin Operator

A `HelmRelease` installs the Spin operator itself, which uses an
`OCIRepository` as the source for installation. If you used Flux versions
earlier than 2.3, you may remember `HelmRepository` - if you have OCI sources
for your Helm repository, you can now use `OCIRepository` instead.

Like `cert-manager` we have enabled automation with a wildcard `~0.2.0` that
should automatically install any new PATCH release.

In the context of pre-releases before 1.0.0, every MINOR bump can include
breaking changes, so we probably should not opt into automatic upgrades without
reading the Changelog first. (But we can install patches!)

This is a way to deploy release upgrades automatically that remains somewhat
conservative. You may prefer to pin versions and bump them manually, or use
some automation to create a Pull Request to alert you when upgrades become
available. Dependabot and Renovate both provide great solutions that are free.

(There's also Flux `ImageUpdateAutomation` which I have covered in many demos
before, I think we maybe won't be using today, but it's worth a mention here!)

The Spin operator is the business end of a `SpinApp`. It translates our request
to run the Spin app into Kubernetes deployment mumbo-jumbo.

Again, abstractions! We don't need users to know everything.

##### Shim Executor

Part of the Spin Operator. `spin-operator.shim-executor.yaml` tells Spin
Operator which namespaces are allowed to deploy `SpinApp` resources, and
perhaps more importantly, what executor type should be used.

We need one in the namespace where we will deploy our `SpinApp`. (We will use
the `default` namespace.)

##### KWasm-operator

Since we have KubeVirt to create virtual machines for our nodes, we can install
our own shim-spin for containerd.

If you are using Talos, you'll use an extension instead as mentioned in the
[tl;dw](#tldw).

#### Applications

If you bootstrapped with us and all of the Flux workloads were suspended, it's
time to unpause them all, one-by-one or at-once, and see the sample apps run.

##### Sample Application

We can see the Quickstart's `simple-spinapp` sample in the default namespace.

It runs from an OCI image, like every `SpinApp` in SpinKube!

##### Flux Docs OCI

It wouldn't be a very good test if we didn't have a little stress. Let's see if
we can break SpinKube!

Continue following along with the demo by cloning
[kingdon-ci/flux-docs](https://github.com/kingdon-ci/flux-docs) and enable
GitHub Actions workflows to try to deploy your own Spin app!

(We will deploy it in our demo using Flux OCI, you can edit the manifest in
your fork to point it at your own images that you built!)

## Thanks

If we manage to show all this stuff in less than an hour, it'll be a miracle...
or maybe just practice?

###### Please Clap
