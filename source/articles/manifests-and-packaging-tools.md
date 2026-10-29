# Manifests, Helm and Kustomize

**Covers:** Manifest, YAML, CSV, Helm, Helm chart, Kustomize

## Describing the wanted state in files {#overview}
Imagine you build a cluster's contents by typing commands one after another. A week later, nobody can repeat what you did, and nobody can review it. Kubernetes avoids this by working from files that describe what you want. You keep the files in version control, and the cluster makes itself match them. This article starts with the file format, then the manifest itself, and ends with two tools for the case in which one application must be installed many times with small differences.

## YAML {#yaml}
Kubernetes files are written in YAML, a text format that people can read easily. It shows nesting by indentation, marks list items with a dash and writes key-value pairs with a colon. Indentation matters here: use two spaces, never tabs, because a wrong level either changes the meaning or breaks the file.

```yaml
name: shop
ports:
  - 80
  - 443
```

## CSV {#csv}
CSV, comma-separated values, is a different format, and you will see it around clusters even though Kubernetes does not use it. Each line is a row, and commas separate the cells. It is the plainest way to exchange spreadsheet data, and many tools can read it. Reports, exports of resource use and lists of settings often arrive as CSV. Unlike YAML, it has no nesting, which is why it only suits tables.

## Manifest {#manifest}
A manifest is a YAML file that describes the state you want. It lists what should exist and leaves out the steps to get there. You hand it to the cluster with `kubectl apply -f shop.yaml`, and Kubernetes works out what to create, change or leave alone.

That is the declarative approach. Applying the same file twice is safe, because the second time nothing changes. A manifest usually has four parts: `apiVersion`, `kind`, `metadata` and `spec`. One file may hold several objects, separated by lines of three dashes.

## Helm {#helm}
A real application needs many manifests: a Deployment, a Service, configuration, storage. Typing them for every installation gets tedious. Helm is a packaging tool that bundles them into one package and fills the gaps with values you supply, so one command, `helm install`, installs the whole application. It also records each installation as a release, so that it can upgrade or undo it later.

## Helm chart {#helm-chart}
The package itself is a Helm chart: a folder with templates of manifests, a file of default values and a description. The templates contain placeholders, such as `{{ .Values.replicas }}`, which Helm fills in from the values. Public charts exist for common software, such as databases and monitoring tools. You install one and override only the values that differ, for instance `--set replicas=5`.

## Kustomize {#kustomize}
Kustomize takes another route: it customizes plain manifests by layering small patches on a base version, with no templating language. You keep one base set of files and, for each environment, a short overlay that says what differs: more replicas in production, another image tag in testing. It is built into `kubectl`, as `kubectl apply -k`.

Here is the trade-off against Helm. With Kustomize, every file stays valid YAML that you can read directly, but it cannot do loops or conditions. Helm is more powerful and more complex.

## A worked example {#example}
Let's take one application and two environments, first with Kustomize.

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

You apply it with `kubectl apply -k overlays/prod`. The result is the same base, with 5 replicas. The test overlay might leave it at 1. Only one short file of about 8 lines differs, instead of two full copies of the base that would drift apart.

Now the same case with Helm. The chart has `replicas: {{ .Values.replicas }}` in its template, and you keep two small value files:

```bash
helm install shop-test ./shop -f values-test.yaml
helm install shop-prod ./shop -f values-prod.yaml
helm upgrade shop-prod ./shop --set replicas=6
helm rollback shop-prod 1
```

Let's count the files. With 3 environments and 10 manifests, copying everything gives 30 files to keep in step. With either tool you have 10 shared files, plus 3 short ones that hold only the differences.

Which should you choose? For your own application with a few variations, Kustomize is enough. For software that other people publish, with many options, use their Helm chart.

## A common mistake {#mistake}
A common slip is wrong indentation in YAML. It either fails with a parse error or, worse, silently changes the meaning by moving a field to another level. Check your files with `kubectl apply --dry-run=client -f shop.yaml`.

Another slip is editing live objects by hand, so the file no longer says what is actually in the cluster.

## Key points {#key-points}
- A manifest is a YAML file that declares the wanted state, and `kubectl apply` makes the cluster match it.
- YAML uses indentation for structure. CSV is a flat table of comma-separated cells.
- A Helm chart is a template package with values. Helm installs, upgrades and undoes releases.
- Kustomize layers patches on a plain base, with no templates.
