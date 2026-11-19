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

## Collection-only 1.0.0 candidate

This branch pins the validated Nokia collection commit
`ca2b292c3e147c219fb4c9fd911e9994e61811b2` (1.0.0 preparation). **There is no
published 1.0.0 collection tag/release yet.** `requirements.yml` pins the commit,
not a nonexistent release tag, for subsequent full rebuilds.

`Dockerfile.collection-release` retains the tested runtime digest from tag
`628f43b` and replaces only the Nokia collection. Other Python packages,
collections and the SSH stack are not upgraded. The previous 0.3.13 EE source
remains available at commit `9005bca`.

Build the collection archive from a clean checkout of the pinned commit. Use a
minimal private build context containing only the archive and checksum; do not
include inventories, credentials, private captures or a whole project checkout.

```sh
COLLECTION_SOURCE=/path/to/nokia.isam-checkout
test "$(git -C "$COLLECTION_SOURCE" rev-parse HEAD)" = ca2b292c3e147c219fb4c9fd911e9994e61811b2
test -z "$(git -C "$COLLECTION_SOURCE" status --porcelain)"
BUILD_CONTEXT=$(mktemp -d)
(cd "$COLLECTION_SOURCE" && ansible-galaxy collection build --output-path "$BUILD_CONTEXT")
(cd "$BUILD_CONTEXT" && sha256sum nokia-isam-1.0.0.tar.gz > nokia-isam-1.0.0.tar.gz.sha256)
IMAGE="ghcr.io/blue-networks/yplay-custom-ee:$(git rev-parse --short HEAD)"
docker build --network none -f Dockerfile.collection-release \
  -t "$IMAGE" "$BUILD_CONTEXT"
```

The Docker build checks the archive's checksum before installation and labels the
image with the collection version and source revision. Verify the installed
collection and run offline module smoke tests before any rollout. For a local
non-root smoke test, use a writable temporary home rather than inherited root-owned
Ansible build caches:

```sh
docker run --rm --network none --tmpfs /tmp:rw,mode=1777 -e HOME=/tmp \
  --entrypoint ansible-galaxy "$IMAGE" collection list nokia.isam
```

Building this candidate does not publish it or update AWX/NOC. Publication and
deployment require a separate decision and review of the collection's 1.0 upgrade
guide. Do not overwrite an existing image tag or assume the major schema changes
are safe for unchanged playbooks.

### Offline validation (2026-11-19)

The collection-only candidate was built from the verified archive of `ca2b292c`
and tested as UID 1000 with networking disabled and no host inventory/credential
mounts. All 609 installed collection file checksums matched the archive. All 72
module documentation entries and the connection documentation loaded successfully;
seven module-entrypoint cases covered generic PON, SIP, VLAN rendering/parsing,
security settings and the new statistics action in check mode. The SSH/libssh
backend and connection plugin imported successfully; no device session was opened.

Compared with the previous `9005bca` EE, all non-Nokia collection versions and the
ten checked Python package versions were unchanged, including Ansible Core
2.19.12, Runner 2.4.3, ansible-pylibssh 1.4.0, anytree 2.13.0 and Paramiko 5.0.0.
The actual network-resource base loaded from the explicit ansible.netcommon 8.6.1
installation rather than the older collection bundled with the Ansible package.
This records local image validation only, not a registry push or AWX rollout.
