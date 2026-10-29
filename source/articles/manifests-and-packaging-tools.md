# Manifests, Helm and Kustomize

**Covers:** Manifest, YAML, CSV, Helm, Helm chart, Kustomize

## Describing the wanted state in files {#overview}
You could build a cluster's contents by typing commands one after another, but nobody could repeat or review that. Kubernetes instead works from files that describe what you want. You keep the files in version control, and the cluster makes itself match them. Two further tools handle the case in which the same application must be installed many times with small differences.

## YAML {#yaml}
YAML is a human-readable text format for structured data. It uses indentation to show nesting, a dash for list items and a colon for key-value pairs. Kubernetes files are written in it. Indentation matters: two spaces, never tabs, and a wrong level changes the meaning or breaks the file.

```yaml
name: shop
ports:
  - 80
  - 443
```

## CSV {#csv}
CSV, or comma-separated values, is a plain-text table format in which each line is a row, and commas separate the cells. It is the simplest way to exchange spreadsheet data, and many tools can read it. It is not a Kubernetes format, but it appears around clusters: reports, exports of resource use and lists of settings often come as CSV. Unlike YAML, it has no nesting, which is why it suits tables only.

## Manifest {#manifest}
A manifest is a YAML file that describes the state you want a cluster to be in. It states what should exist, not the steps to get there. You then give it to the cluster with `kubectl apply -f shop.yaml`, and Kubernetes works out what to create, change or leave alone.

This is the declarative approach. Applying the same file twice is safe: the second time, nothing changes. A manifest has four usual parts: `apiVersion`, `kind`, `metadata` and `spec`. One file may hold several objects, separated by lines of three dashes.

## Helm {#helm}
Helm is a packaging tool for Kubernetes applications. A real application needs many manifests: a Deployment, a Service, configuration, storage. Helm bundles them into one package and fills the gaps with values that you supply, so that you can install the whole application with a single command, `helm install`. Helm also keeps a record of each installation, called a release, so it can upgrade or undo it.

## Helm chart {#helm-chart}
A Helm chart is the package itself: a folder with templates of manifests, a file of default values and a description. The templates contain placeholders, such as `{{ .Values.replicas }}`, which Helm fills in from the values. Public charts exist for common software, such as databases and monitoring tools. You install one and override only the values that differ, for instance `--set replicas=5`.

## Kustomize {#kustomize}
Kustomize customizes plain manifests by layering small patches on top of a base version, without any templating language. You keep one base set of files and, for each environment, a short overlay that says what differs: more replicas in production, another image tag in testing. It is built into `kubectl`, with `kubectl apply -k`.

The trade-off against Helm: Kustomize keeps every file valid YAML you can read directly, but it cannot do loops or conditions. Helm is more powerful and more complex.

## A worked example {#example}
One application, two environments, shown with Kustomize.

```text
shop/
  base/
    deployment.yaml
    kustomization.yaml
  overlays/
    test/kustomization.yaml
    prod/kustomization.yaml
```

The base Deployment asks for 1 replica. The production overlay is:

```yaml
resources:
  - ../../base
patches:
  - target:
      kind: Deployment
      name: shop-web
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
```

Apply with `kubectl apply -k overlays/prod`. The result: the same base, with 5 replicas. The test overlay might leave it at 1. Only 1 file differs by about 8 lines, instead of 2 full copies of the base that would drift apart.

The same case with Helm. The chart has `replicas: {{ .Values.replicas }}` in its template, and two small value files:

```bash
helm install shop-test ./shop -f values-test.yaml
helm install shop-prod ./shop -f values-prod.yaml
helm upgrade shop-prod ./shop --set replicas=6
helm rollback shop-prod 1
```

Count the files. With 3 environments and 10 manifests, copying gives 30 files to keep in step. With either tool, there are 10 shared files, plus 3 short ones that hold only the differences.

Which to choose: for your own application with a few variations, Kustomize is enough. For software that other people publish and that has many options, use their Helm chart.

## A common mistake {#mistake}
Wrong indentation in YAML, which either fails with a parse error or, worse, silently changes the meaning by moving a field to another level. Check files with `kubectl apply --dry-run=client -f shop.yaml`. Another mistake is editing live objects by hand, so the file no longer says what is running.

## Key points {#key-points}
- A manifest is a YAML file that declares the wanted state. `kubectl apply` makes the cluster match it.
- YAML uses indentation for structure. CSV is a flat table of comma-separated cells.
- A Helm chart is a template package with values. Helm installs, upgrades and undoes releases.
- Kustomize layers patches on a plain base, with no templates.
