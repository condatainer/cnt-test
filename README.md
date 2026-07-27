# cnt-test

Small recipe collection for exercising CondaTainer builds and OCI registry
distribution. Its descriptor uses the internal endpoint
`ghcr.io/condatainer/cnt-test`, which permits every test artifact type.

| Artifact | Type | Purpose |
|---|---|---|
| `ubuntu24/base` | base | Ubuntu 24.04 with Apptainer 1.5.2, micromamba 2.6.2-1, and SquashFS tools |
| `ubuntu24/simple-os` | os | Small definition-built OS layer containing `jq` |
| `ubuntu24/versioned-os/1.0` | os | Versioned OS producing OCI repository `ubuntu24/versioned-os`, tag `1.0` |
| `hello/1.0` | app | `noarch` command named `cnt-test-hello` |
| `template-message/{red,blue}` | app | One template recipe whose selected placeholder changes its equivalence SHA |
| `testdata/plain/1.0` | data | Standalone `noarch` text payload |
| `testdata/combined/1.0` | data | Depends on the app and plain data artifact |

## Use as a local source

Configure the absolute path to this directory as a source, then build the
dependency-backed root:

```text
condatainer config prepend sources test=/absolute/path/to/cnt-test
condatainer create testdata/combined/1.0
```

That covers dependency resolution and produces all three script-backed
artifacts. Build the definition-backed cases separately:

```text
condatainer update --base
condatainer create simple-os
condatainer create versioned-os/1.0
```

With `base: ubuntu24`, create first checks each name exactly and then tries the
`ubuntu24/` prefix for names containing at most one slash. Thus the two commands
above resolve to `ubuntu24/simple-os` and `ubuntu24/versioned-os/1.0`.

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
condatainer registry push testdata/combined/1.0
```

Each push infers `ghcr.io/condatainer/cnt-test` and `internal` visibility from
the artifact's recorded source and `source.json`. Add `--registry` to override
the inferred destination.

Remove the local images and run `create` again to exercise automatic prebuilt
acquisition. The combined data artifact tests both dependency forms: app
equivalence by name and data equivalence by its recorded equivalence key.
