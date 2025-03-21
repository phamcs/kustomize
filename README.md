# kustomize

`kustomize` lets you customize raw, template-free YAML
files for multiple purposes, leaving the original YAML
untouched and usable as is.

`kustomize` targets kubernetes; it understands and can
patch [kubernetes style] API objects.  It's like
[`make`], in that what it does is declared in a file,
and it's like [`sed`], in that it emits edited text.

This tool is sponsored by [sig-cli] ([KEP]).

 - [Installation instructions](https://kubectl.docs.kubernetes.io/installation/kustomize/)
 - [General documentation](https://kubectl.docs.kubernetes.io/references/kustomize/)
 - [Examples](examples)

[![Build Status](https://prow.k8s.io/badge.svg?jobs=kustomize-presubmit-master)](https://prow.k8s.io/job-history/kubernetes-jenkins/pr-logs/directory/kustomize-presubmit-master)
[![Go Report Card](https://goreportcard.com/badge/github.com/kubernetes-sigs/kustomize)](https://goreportcard.com/report/github.com/kubernetes-sigs/kustomize)

## kubectl integration

To find the kustomize version embedded in recent versions of kubectl, run `kubectl version`:

```sh
> kubectl version --client
Client Version: v1.31.0
Kustomize Version: v5.4.2
```

The kustomize build flow at [v2.0.3] was added
to [kubectl v1.14][kubectl announcement].  The kustomize
flow in kubectl remained frozen at v2.0.3 until kubectl v1.21,
which [updated it to v4.0.5][kust-in-kubectl update]. It will
be updated on a regular basis going forward, and such updates
will be reflected in the Kubernetes release notes.

| Kubectl version | Kustomize version |
| --------------- | ----------------- |
| < v1.14         | n/a               |
| v1.14-v1.20     | v2.0.3            |
| v1.21           | v4.0.5            |
| v1.22           | v4.2.0            |
| v1.23           | v4.4.1            |
| v1.24           | v4.5.4            |
| v1.25           | v4.5.7            |
| v1.26           | v4.5.7            |
| v1.27           | v5.0.1            |

[v2.0.3]: https://github.com/kubernetes-sigs/kustomize/releases/tag/v2.0.3
[#2506]: https://github.com/kubernetes-sigs/kustomize/issues/2506
[#1500]: https://github.com/kubernetes-sigs/kustomize/issues/1500
[kust-in-kubectl update]: https://github.com/kubernetes/kubernetes/blob/4d75a6238a6e330337526e0513e67d02b1940b63/CHANGELOG/CHANGELOG-1.21.md#kustomize-updates-in-kubectl

For examples and guides for using the kubectl integration please
see the [kubernetes documentation].

## Usage


### 1) Make a [kustomization] file

In some directory containing your YAML [resource]
files (deployments, services, configmaps, etc.), create a
[kustomization] file.

This file should declare those resources, and any
customization to apply to them, e.g. _add a common
label_.

```

base: kustomization + resources

kustomization.yaml                                      deployment.yaml                                                 service.yaml
+---------------------------------------------+         +-------------------------------------------------------+       +-----------------------------------+
| apiVersion: kustomize.config.k8s.io/v1beta1 |         | apiVersion: apps/v1                                   |       | apiVersion: v1                    |
| kind: Kustomization                         |         | kind: Deployment                                      |       | kind: Service                     |
| labels:                                     |         | metadata:                                             |       | metadata:                         |
| - includeSelectors: true                    |         |   name: myapp                                         |       |   name: myapp                     |
|   pairs:                                    |         | spec:                                                 |       | spec:                             |
|     app: myapp                              |         |   selector:                                           |       |   selector:                       |
| resources:                                  |         |     matchLabels:                                      |       |     app: myapp                    |
|   - deployment.yaml                         |         |       app: myapp                                      |       |   ports:                          |
|   - service.yaml                            |         |   template:                                           |       |     - port: 6060                  |
| configMapGenerator:                         |         |     metadata:                                         |       |       targetPort: 6060            |
|   - name: myapp-map                         |         |       labels:                                         |       +-----------------------------------+
|     literals:                               |         |         app: myapp                                    |
|       - KEY=value                           |         |     spec:                                             |
+---------------------------------------------+         |       containers:                                     |
                                                        |         - name: myapp                                 |
                                                        |           image: myapp                                |
                                                        |           resources:                                  |
                                                        |             limits:                                   |
                                                        |               memory: "128Mi"                         |
                                                        |               cpu: "500m"                             |
                                                        |           ports:                                      |
                                                        |             - containerPort: 6060                     |
                                                        +-------------------------------------------------------+

```

File structure:

> ```
> ~/someApp
> ├── deployment.yaml
> ├── kustomization.yaml
> └── service.yaml
> ```

The resources in this directory could be a fork of
someone else's configuration.  If so, you can easily
rebase from the source material to capture
improvements, because you don't modify the resources
directly.

Generate customized YAML with:

```
kustomize build ~/someApp
```

The YAML can be directly [applied] to a cluster:

> ```
> kustomize build ~/someApp | kubectl apply -f -
> ```


### 2) Create [variants] using [overlays]

Manage traditional [variants] of a configuration - like
_development_, _staging_ and _production_ - using
[overlays] that modify a common [base].

```

overlay: kustomization + patches

kustomization.yaml                                      replica_count.yaml                      cpu_count.yaml
+-----------------------------------------------+       +-------------------------------+       +------------------------------------------+
| apiVersion: kustomize.config.k8s.io/v1beta1   |       | apiVersion: apps/v1           |       | apiVersion: apps/v1                      |
| kind: Kustomization                           |       | kind: Deployment              |       | kind: Deployment                         |
| labels:                                       |       | metadata:                     |       | metadata:                                |
|  - includeSelectors: true                     |       |   name: myapp                 |       |   name: myapp                            |
|    pairs:                                     |       | spec:                         |       | spec:                                    |
|      variant: prod                            |       |   replicas: 80                |       |  template:                               |
| resources:                                    |       +-------------------------------+       |     spec:                                |
|   - ../../base                                |                                               |       containers:                        |
| patches:                                      |                                               |         - name: myapp                    |
|   - path: replica_count.yaml                  |                                               |           resources:                     |
|   - path: cpu_count.yaml                      |                                               |             limits:                      |
+-----------------------------------------------+                                               |               memory: "128Mi"            |
                                                                                                |               cpu: "7000m"               |
                                                                                                +------------------------------------------+
```


File structure:
> ```
> ~/someApp
> ├── base
> │   ├── deployment.yaml
> │   ├── kustomization.yaml
> │   └── service.yaml
> └── overlays
>     ├── development
>     │   ├── cpu_count.yaml
>     │   ├── kustomization.yaml
>     │   └── replica_count.yaml
>     └── production
>         ├── cpu_count.yaml
>         ├── kustomization.yaml
>         └── replica_count.yaml
> ```

Take the work from step (1) above, move it into a
`someApp` subdirectory called `base`, then
place overlays in a sibling directory.

An overlay is just another kustomization, referring to
the base, and referring to patches to apply to that
base.

This arrangement makes it easy to manage your
configuration with `git`.  The base could have files
from an upstream repository managed by someone else.
The overlays could be in a repository you own.
Arranging the repo clones as siblings on disk avoids
the need for git submodules (though that works fine, if
you are a submodule fan).

Generate YAML with

```sh
kustomize build ~/someApp/overlays/production
```

The YAML can be directly [applied] to a cluster:

> ```sh
> kustomize build ~/someApp/overlays/production | kubectl apply -f -
> ```

## Community

- [file a bug](https://kubectl.docs.kubernetes.io/contributing/kustomize/bugs/)
- [contribute a feature](https://kubectl.docs.kubernetes.io/contributing/kustomize/features/)
- [propose a larger enhancement](https://github.com/kubernetes-sigs/kustomize/tree/master/proposals)

### Code of conduct

Participation in the Kubernetes community
is governed by the [Kubernetes Code of Conduct].

[`make`]: https://www.gnu.org/software/make
[`sed`]: https://www.gnu.org/software/sed
[DAM]: https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#declarative-application-management
[KEP]: https://github.com/kubernetes/enhancements/blob/master/keps/sig-cli/2377-Kustomize/README.md
[Kubernetes Code of Conduct]: code-of-conduct.md
[applied]: https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#apply
[base]: https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#base
[declarative configuration]: https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#declarative-application-management
[kubectl announcement]: https://kubernetes.io/blog/2019/03/25/kubernetes-1-14-release-announcement
[kubernetes documentation]: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
[kubernetes style]: https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#kubernetes-style-object
[kustomization]: https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#kustomization
[overlay]: https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#overlay
[overlays]: https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#overlay
[release page]: https://github.com/kubernetes-sigs/kustomize/releases
[resource]: https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#resource
[resources]: https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#resource
[sig-cli]: https://github.com/kubernetes/community/blob/master/sig-cli/README.md
[variants]: https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#variant
