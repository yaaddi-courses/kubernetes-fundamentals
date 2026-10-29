# ConfigMaps and Secrets

**Covers:** ConfigMap, Secret, README

## Keeping settings out of the image {#overview}
An application needs settings: the address of the database, a feature switch, a password. If they are baked into the container image, then changing one means building and releasing a new image, and the same image cannot be used in testing and in production. The cure is to keep configuration outside the image and supply it when the pod starts. Kubernetes has two objects for this, one for ordinary settings and one for sensitive ones.

## README {#readme}
A README is the first document of a project, kept at the top of its folder, telling a newcomer what the project is for and how to start. For configuration objects, a README is where you document each setting: what it does, what values are allowed and where its value comes from. Because configuration is spread over many objects, a short document saves a new colleague from searching.

## ConfigMap {#configmap}
A ConfigMap is an object that holds an application's configuration separately from its container image, as key-value pairs or as whole files, so that you can change a setting without rebuilding the image. A ConfigMap is not secret and is stored in plain form, so it suits things like a log level, a URL or the contents of a configuration file.

A pod can use it in two ways: as environment variables, or as files mounted into the container's file system. Mounted files update in place when the ConfigMap changes, after a short delay, but environment variables are read only when the pod starts, so the pod needs a restart to see a new value.

## Secret {#secret}
A Secret is like a ConfigMap, but for sensitive values such as passwords, tokens and keys, handled with more care so that they do not sit in plain view. It is used in the same ways, as environment variables or mounted files. Kubernetes can restrict who may read Secrets, keeps them out of ordinary listings and sends them only to the nodes that need them.

An important warning: by default, the values in a Secret are only encoded with base64, which is not encryption. Anyone who can read the Secret object can decode it. For real protection, turn on encryption at rest for etcd, limit who can read Secrets with access rules and consider an external secret store.

## A worked example {#example}
A ConfigMap and a Secret used by one pod:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: shop-config
data:
  LOG_LEVEL: "info"
  DB_HOST: "shop-db"
---
apiVersion: v1
kind: Secret
metadata:
  name: shop-secret
type: Opaque
stringData:
  DB_PASSWORD: "change-me-example"
---
apiVersion: v1
kind: Pod
metadata:
  name: shop-web
spec:
  containers:
    - name: web
      image: shop/web:1.4
      envFrom:
        - configMapRef:
            name: shop-config
        - secretRef:
            name: shop-secret
```

What happens:

1. The pod starts, and Kubernetes fills the container's environment with `LOG_LEVEL`, `DB_HOST` and `DB_PASSWORD`.
2. The same image can be used in two environments. In production, the ConfigMap has `LOG_LEVEL: "warn"` and a different `DB_HOST`. In testing, it has `info` and a test database. The image is unchanged.
3. To change the log level, edit the ConfigMap and restart the pod, with no image build.

Check how the password is stored:

```text
echo -n "change-me-example" | base64
Y2hhbmdlLW1lLWV4YW1wbGU=
```

Anyone can reverse that with `base64 --decode`, so the encoding is not protection. Real protection comes from restricting access.

Count the benefit. A team with 3 environments and no ConfigMap needs 3 images to differ in a few settings, and every settings change means a build, a test and a release, perhaps 30 minutes. With a ConfigMap, it is 1 image and a 1-minute edit. Over 20 settings changes a year, that saves about 10 hours.

Documentation helps too. A README in the repository lists the keys: `LOG_LEVEL` (one of debug, info, warn), `DB_HOST` (the database Service name) and `DB_PASSWORD` (from the Secret `shop-secret`). A new engineer sets up a test environment in minutes.

## A common mistake {#mistake}
Putting a password in a ConfigMap, or committing a Secret file with real values to the source repository. Both expose it. Keep real secrets out of Git, or encrypt them. Another mistake is changing a ConfigMap and expecting an environment variable to update without a restart.

## Key points {#key-points}
- A ConfigMap holds ordinary settings outside the image. A Secret holds sensitive values.
- Both can be given to a pod as environment variables or as mounted files.
- Secret values are only base64-encoded, not encrypted. Restrict access and enable encryption.
- Document each setting in a README.
