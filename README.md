# Yplay Custom Execution Environment

Execution environment which enables the execution of the playbooks in our repositories.

Install the pinned build tool and authenticate Docker to GHCR first:

```bash
python3 -m pip install -r requirements-build.txt
docker login ghcr.io
```

Build and publish an immutable image tag matching the source commit:

```bash
TAG=$(git rev-parse --short HEAD)
IMAGE="ghcr.io/blue-networks/yplay-custom-ee:${TAG}"
ansible-builder build --container-runtime docker -t "$IMAGE"
docker push "$IMAGE"
```

After publishing, update the AWX Execution Environment image to the new tag. The
Yplay Customer Reconcile v2 templates use the registered Yplay EE, which is
currently pinned to a commit-specific image tag.

## Collection-only 0.3.10 rollout

`Dockerfile.collection-release` preserves the deployed EE digest and changes only
the Nokia collection. Download the release artifact into a private build context,
verify it against the published checksum, then build with this Dockerfile and tag
the image with the EE source commit. Do not add credentials or inventory to the
build context. `requirements.yml` also pins 0.3.10 for subsequent full rebuilds.

```sh
gh release download 0.3.10 --repo jahknem/nokia.isam --pattern 'nokia-isam-0.3.10.tar.gz*' --dir "$BUILD_CONTEXT"
(cd "$BUILD_CONTEXT" && sha256sum -c nokia-isam-0.3.10.tar.gz.sha256)
docker build -f Dockerfile.collection-release -t "ghcr.io/blue-networks/yplay-custom-ee:$(git rev-parse --short HEAD)" "$BUILD_CONTEXT"
```
