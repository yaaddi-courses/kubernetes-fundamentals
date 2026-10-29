# ConfigMaps and Secrets

**Covers:** ConfigMap, Secret, README

## Keeping settings out of the image {#overview}
Every application needs settings: the address of the database, a feature switch, a password. If you bake them into the container image, changing one means building and releasing a new image, and you cannot use the same image in testing and in production. The cure is to keep configuration outside the image and hand it over when the pod starts. Kubernetes has two objects for this, one for ordinary settings and one for sensitive ones. We will also look at the document that explains the settings to your colleagues.

## README {#readme}
Let's begin with the humble README. It is the first document of a project, kept at the top of its folder, telling a newcomer what the project is for and how to start. For configuration, the README is where you write down each setting: what it does, which values are allowed and where its value comes from. Configuration is spread over many objects, so a short document saves a new colleague from searching.

## ConfigMap {#configmap}
A ConfigMap holds an application's configuration apart from its container image. It stores key-value pairs or whole files, and you can change a setting without rebuilding the image. A ConfigMap is not secret and is stored in plain form, so it suits a log level, a URL or the contents of a configuration file.

A pod can use it in two ways: as environment variables, or as files mounted into the container's file system. The two behave differently when you edit the ConfigMap. Mounted files update in place after a short delay. Environment variables are read only when the pod starts, so the pod needs a restart to see a new value.

## Secret {#secret}
A Secret looks like a ConfigMap, but it is meant for sensitive values: passwords, tokens and keys. Kubernetes handles it with more care so that it does not sit in plain view. You use it in the same two ways, as environment variables or as mounted files. Kubernetes can restrict who may read Secrets, keeps them out of ordinary listings and sends them only to the nodes that need them.

Here is a warning you should remember. By default, the values in a Secret are only encoded with base64, and that is not encryption. Anyone who can read the Secret object can decode it. For real protection, turn on encryption at rest for etcd, limit who can read Secrets with access rules, and consider an external secret store.

## A worked example {#example}
Let's look at a ConfigMap and a Secret used by one pod:

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

Here is what happens:

1. The pod starts, and Kubernetes fills the container's environment with `LOG_LEVEL`, `DB_HOST` and `DB_PASSWORD`.
2. The same image works in two environments. In production, the ConfigMap has `LOG_LEVEL: "warn"` and a different `DB_HOST`. In testing, it has `info` and a test database. The image never changes.
3. To change the log level, you edit the ConfigMap and restart the pod, with no image build.

Now check how the password is stored:

```text
echo -n "change-me-example" | base64
Y2hhbmdlLW1lLWV4YW1wbGU=
```

Anyone can reverse that with `base64 --decode`, so the encoding is not protection. Real protection comes from restricting access.

Now count the benefit. A team with 3 environments and no ConfigMap needs 3 images that differ in a few settings, and every change means a build, a test and a release, perhaps 30 minutes. With a ConfigMap, it is 1 image and a 1-minute edit. Over 20 settings changes a year, you save about 10 hours.

The README helps too. It lists the keys: `LOG_LEVEL` (one of debug, info, warn), `DB_HOST` (the database Service name) and `DB_PASSWORD` (from the Secret `shop-secret`). A new engineer can prepare a test environment in minutes.

## A common mistake {#mistake}
A common slip is putting a password in a ConfigMap, or committing a Secret file with real values to the source repository. Both expose the password. Keep real secrets out of Git, or encrypt them first.

Another slip is changing a ConfigMap and expecting an environment variable to update without a restart. It will not, because the value was read once at startup.

## Key points {#key-points}
- A ConfigMap holds ordinary settings outside the image. A Secret holds sensitive values.
- Both can reach a pod as environment variables or as mounted files.
- Secret values are only base64-encoded, not encrypted. Restrict access and enable encryption.
- Document each setting in a README.
