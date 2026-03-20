# Fix pause:3.9 Layer Format

## Problem

The `pause:3.9` image on the default registry (`47.79.87.161:8968`) uses Docker manifest format. Its layer has media type `application/vnd.docker.image.rootfs.diff.tar.gzip`.

rkforge's `get_media_type()` in `rkforge/src/pull/media.rs` only recognizes:
- `tar+gzip` → TarGzip (decompresses + untars)
- `tar` → Tar (untars)
- anything else → Other (**copies as-is**)

Since Docker's `tar.gzip` doesn't match `tar+gzip`, the layer is copied as a raw gzip file instead of being unpacked to a directory. When `mount_and_copy_bundle` tries to create an overlay filesystem using this file as a lowerdir, the merged directory is empty and `cp -a merged/* rootfs/` fails.

Images pushed in OCI format (like `nginx:latest` on the same registry) use `application/vnd.oci.image.layer.v1.tar+gzip` and are correctly unpacked.

## Fix Script

Run this after `rkforge pull pause:3.9`:

```bash
#!/bin/bash
set -e

REG="http://47.79.87.161:8968"

# 1. Get the layer digest from the manifest
MANIFEST=$(curl -s "$REG/v2/library/pause/manifests/3.9" \
  -H "Accept: application/vnd.oci.image.manifest.v1+json, application/vnd.docker.distribution.manifest.v2+json")

LAYER_HASH=$(echo "$MANIFEST" | python3 -c "
import sys, json
m = json.load(sys.stdin)
print(m['layers'][0]['digest'].split(':')[1])
")

echo "Layer hash: $LAYER_HASH"

LAYER_PATH="/var/lib/rkforge/layers/$LAYER_HASH"

# 2. Check if it's already a directory (already fixed or OCI format)
if sudo test -d "$LAYER_PATH"; then
    echo "Layer is already a directory, no fix needed"
    exit 0
fi

# 3. Check if it's a file (needs fixing)
if sudo test -f "$LAYER_PATH"; then
    FILE_TYPE=$(sudo file "$LAYER_PATH" | cut -d: -f2)
    echo "Layer is a file: $FILE_TYPE"

    # Rename, unpack, cleanup
    sudo mv "$LAYER_PATH" "${LAYER_PATH}.gz"
    sudo mkdir -p "$LAYER_PATH"
    sudo tar xzf "${LAYER_PATH}.gz" -C "$LAYER_PATH"
    sudo rm "${LAYER_PATH}.gz"

    # Verify
    if sudo test -d "$LAYER_PATH"; then
        echo "Fix successful: layer unpacked to directory"
        sudo ls "$LAYER_PATH" | head -5
    else
        echo "ERROR: Fix failed"
        exit 1
    fi
else
    echo "Layer not found at $LAYER_PATH — was rkforge pull run?"
    exit 1
fi
```

## Verification

After fixing, the layer path should be a directory containing the pause rootfs:

```bash
$ sudo ls /var/lib/rkforge/layers/<hash>/
dev  etc  pause
```

And `pause` should have execute permissions:

```bash
$ sudo ls -la /var/lib/rkforge/layers/<hash>/pause
-rwxr-xr-x 1 root root 743952 ... pause
```

If `pause` lacks execute permission (e.g., built from a local bundle where it wasn't `chmod +x`), the container will fail with `executable '/pause' does not have correct permissions`.
