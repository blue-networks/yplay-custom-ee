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
