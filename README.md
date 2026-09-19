# cnt-test

Small recipe collection for exercising CondaTainer builds and OCI registry
distribution. Its descriptor publishes to the `public` endpoint
`ghcr.io/condatainer/cnt-test`.

| Artifact | Type | Purpose |
|---|---|---|
| `ubuntu24/base` | os | Minimal container root: `bash`, `e2fsprogs`, `fuse2fs` |
| `ubuntu24/simple-os` | os | Small definition-built OS layer containing `jq` |
| `ubuntu24/versioned-os/1.0` | os | Versioned OS producing OCI repository `ubuntu24/versioned-os`, tag `1.0` |
| `hello/1.0` | app | `noarch` command named `cnt-test-hello` |
| `template-message/{red,blue}` | app | One template recipe whose selected placeholder changes its equivalence SHA |
| `testdata/plain/1.0` | data | Standalone `noarch` text payload |
| `testdata/hello/1.0/combined` | data | Depends on the app and the plain data artifact |
| `testdata/layers/20g` | data | Synthetic 20 GiB payload for multipart OCI-layer testing |

The apps declare `#REDISTRIBUTE:yes`, which a public endpoint requires of an
app; OS and data recipes need no declaration.

## Use as a source

Configure the repository's GitHub `main` branch as a source, then build the
dependency-backed artifact:

```text
condatainer config prepend sources test=https://raw.githubusercontent.com/condatainer/cnt-test/main
condatainer create testdata/hello/1.0/combined
```

For local development, replace the URL with the absolute path to this checkout.

That resolves the dependency graph and builds all three script-backed
artifacts. Build the definition-backed cases separately:

```text
condatainer create base
condatainer create simple-os
condatainer create versioned-os/1.0
```

With `default_distro: ubuntu24`, `create` first checks each name exactly and then
tries the `ubuntu24/` prefix for names containing at most one slash. Thus the
commands above resolve to `ubuntu24/base`, `ubuntu24/simple-os` and
`ubuntu24/versioned-os/1.0`.

## Template equivalence-key test

Build both concrete targets from the same template recipe:

```text
condatainer create template-message/red
condatainer create template-message/blue
```

Each image embeds the same unexpanded `recipes/template-message` source, while
its manifest records the selected `flavor`. Confirm the resulting equivalence
SHA values differ by reading each image's `/.cnt/manifest.json`:

```text
for flavor in red blue; do
  image=/path/to/images/template-message--${flavor}.sqf
  printf '%s: ' "$flavor"
  unsquashfs -cat "$image" .cnt/manifest.json | jq -r '.keys.equiv.sha256'
done
```

The two hashes must be different. Running `cnt-template-message` from each
artifact also prints its selected value (`red` or `blue`).

## Registry smoke test

Authenticate to GHCR, build locally once, and publish each resulting artifact:

```text
condatainer registry login ghcr.io --username USER --password-stdin
condatainer registry push ubuntu24/base
condatainer registry push ubuntu24/simple-os
condatainer registry push ubuntu24/versioned-os/1.0
condatainer registry push hello/1.0
condatainer registry push testdata/plain/1.0
condatainer registry push testdata/hello/1.0/combined
```

Each push infers `ghcr.io/condatainer/cnt-test` and the `public` audience from
the artifact's recorded source and `source.json`. Add `--registry` to override
the inferred destination.

> [!NOTE]
> CondaTainer's endpoint audience is a publication policy; it does not set
> the GHCR package visibility. A package first created by a local push is not
> public even when `cnt-test` is public and **Inherit access from source
> repository** is enabled. Make each distinct GHCR package public once under
> **Package settings → Danger Zone → Change visibility**. Later tags pushed to
> that package retain its public visibility.

Remove the local images and run `create` again to exercise automatic prebuilt
acquisition. The combined data artifact tests both dependency forms: `hello/1.0` appears
in its name, so it counts toward equivalence by name, and `testdata/plain/1.0`
counts by its recorded equivalence key.

## Large multipart-layer test

`testdata/layers/20g` writes an incompressible 20 GiB payload plus a small marker
file. Incompressible bytes are required because zero-filled or sparse input
would collapse to a tiny SquashFS and would not test multipart upload.

Build it only on a machine with enough scratch and image storage, then push it:

```text
condatainer create testdata/layers/20g
condatainer registry push testdata/layers/20g
```

CondaTainer splits artifacts above 512 MiB into ordered OCI layers. The small
SquashFS overhead puts this fixture just above 20 GiB, so it should publish as
about 41 layers. Use the GHCR package page or inspect the manifest to confirm
the layer count, then remove the local image and create it again to test pull
and reassembly:

```text
condatainer create testdata/layers/20g
```

Budget more than 40 GiB of free space for a pull: downloaded chunks and the
reassembled SquashFS coexist until installation completes.
