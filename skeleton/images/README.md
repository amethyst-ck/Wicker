# Derived images

Dockerfiles for **derived images** go here — one subdirectory per service.
Canasta supports custom images for exactly two services: `web/` and
`elasticsearch/`. Additional containers belong in
[`../sidecars/`](../sidecars/), not here.

> Not wiki media. These are Docker images. Files you want *uploaded into the
> wiki* (logos on wiki pages, `File:` uploads) go under
> [`../content/uploads/`](../content/uploads/), not here.

Reach for a derived image only when you need something that can't be added
by dropping files into `extensions/` or by setting an env knob: OS
packages, system libraries, or `php.ini` settings for `web`; an analysis
plugin or alternate collation for `elasticsearch`.

## How it maps to `wicker.yaml`

Each image is one entry under `images:` in
[`../wicker.yaml`](../wicker.yaml), keyed by service name:

```yaml
images:
  web:
    dockerfile: ./images/web/Dockerfile
    context: ./images/web        # default: the Dockerfile's directory
    # baseImage: ghcr.io/canastawiki/canasta:x.y.z   # default: the release the CLI targets
  elasticsearch:
    dockerfile: ./images/elasticsearch/Dockerfile
    # tag: my-es-image:local     # default: derived from app name + service + version
```

Example `images/web/Dockerfile`:

```dockerfile
# Wicker supplies the base via --build-arg BASE_IMAGE, so use this ARG.
ARG BASE_IMAGE=ghcr.io/canastawiki/canasta:latest
FROM ${BASE_IMAGE}

# e.g. raise an upload limit that has no .env knob:
RUN echo "max_file_uploads = 200" > /usr/local/etc/php/conf.d/uploads.ini
```

The base image is supplied at build time as the `BASE_IMAGE` build
argument — from the entry's `baseImage` key, or the `--base-image` deploy
flag (which takes precedence for `web`). So the Dockerfile must declare
`ARG BASE_IMAGE` and `FROM ${BASE_IMAGE}`. For `web` with neither set,
the base is the Canasta release the installed `canasta` CLI targets, and
`canasta upgrade` moves it to each newer release. A `baseImage` pinned to
a Canasta release moves the same way; any other value stays as given.

An `elasticsearch` Dockerfile derives from the stock Elasticsearch image
instead — check the Elasticsearch version your Canasta release runs and
match it in the `FROM`. For example, to add the ICU analysis plugin for
an alternate collation:

```dockerfile
# Match the Elasticsearch version your Canasta release runs.
FROM docker.elastic.co/elasticsearch/elasticsearch:7.10.2
RUN bin/elasticsearch-plugin install --batch analysis-icu
```

## How it's delivered

Both images are delivered on both orchestrators. The `web` image rides
`canasta create --build-from`: Canasta builds it under its own image name
(on Kubernetes it also pushes it to the in-cluster registry), and
rebuilds it from the same tree on `canasta upgrade`. `wicker upgrade`
restages that tree and has Canasta rebuild it, so the `tag` key does not
apply to `web`. Wicker also passes the app version as the
`WICKER_APP_VERSION` build argument, which a Dockerfile may declare (for
example for a `LABEL`). This needs canasta CLI 4.27.0 or later. The
`elasticsearch` image is built locally on
Compose and pushed via `canasta image push` on Kubernetes; either way,
Wicker points the instance's `CANASTA_ELASTICSEARCH_IMAGE` at the
result. On Kubernetes the Dockerfile must sit at the context root.
Re-deploys re-push the same tag; run `canasta image prune` on the
cluster host occasionally to reclaim the superseded layers.
