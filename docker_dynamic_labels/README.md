# Docker Dynamic Tags Bug Demo

This repository demonstrates a bug in Pants where Docker image builds have unstable hashes when the image depends on another Docker image with dynamic `image_tags`.

## Quick Reproduction

```bash
# Clone this repo and run the following commands to see the bug:

# Run 1: Note the hash values in the output
RANDOM_TAG=tag-${RANDOM} pants package ::

# Run 2: Different hash values even though nothing changed!
RANDOM_TAG=tag-${RANDOM} pants package ::

# For clearer demonstration, disable buildx attestations and use fixed tag:
BUILDX_NO_DEFAULT_ATTESTATIONS=1 RANDOM_TAG=tag pants package ::
BUILDX_NO_DEFAULT_ATTESTATIONS=1 RANDOM_TAG=tag pants package ::
# Now the hashes are stable! But change the tag...

BUILDX_NO_DEFAULT_ATTESTATIONS=1 RANDOM_TAG=different-tag pants package ::
# Hashes change even though source code is identical!
```

**Expected:** The `{pants.hash}` values should remain the same since no source files changed.

**Actual:** The hash changes when tags change, even though tags are just metadata.

## Repository Structure

```
.
├── BUILD                 # Default configuration with dynamic tags
├── Dockerfile.base      # Base image dockerfile
├── Dockerfile           # Main image that depends on base
└── README.md           # This file
```

## The Bug

### What's Happening

1. **Base image** (`base`) has dynamic tags including `env("RANDOM_TAG")`
2. **Main image** (`main`) depends on `base` via `FROM ${BASE_IMAGE}` 
3. When building `main`, Pants includes the base image's metadata (including dynamic tags) in the build context
4. This causes `main`'s hash to change even when no actual content has changed

### Files in This Demo

#### `BUILD`
```python
__defaults__(
    {
        "docker_image": dict(
            source="Dockerfile",
            build_platform=["linux/amd64"],
            image_tags=[
                "{pants.hash}",      # This should be stable but isn't
                env("RANDOM_TAG")    # Dynamic tag causing the issue
            ],
        )
    }
)

docker_image(
    name="base",
    source="Dockerfile.base",
)

docker_image(
    name="main",
    source="Dockerfile",
)
```

#### `Dockerfile.base`
```dockerfile
FROM ubuntu
```

#### `Dockerfile`
```dockerfile
ARG BASE_IMAGE=:base
FROM ${BASE_IMAGE}
RUN apt-get update && apt-get install -y python3 python3-pip
```

## Example Output Showing the Problem

### Scenario 1: Dynamic Tags (Using `${RANDOM}`)

#### First Run
```bash
$ RANDOM_TAG=tag-${RANDOM} pants package ::
...
Built docker images: 
  * base:a4ccbb102388059699c1cc41f89157d80da0e2415d1eb7d983542aa8d78dc709
  * base:tag-20723
...
Built docker images: 
  * main:e574c1526c092e2bec410b26163be79e33f7d7a2f7ce58bad9c647d79fc653c8
  * main:tag-20723
```

#### Second Run (immediately after, no changes)
```bash
$ RANDOM_TAG=tag-${RANDOM} pants package ::
...
Built docker images: 
  * base:69bae7a36f3d550a1377d3685d7cca22e7f2617960c756da78b26f0329ec551d  # Different hash!
  * base:tag-4767
...
Built docker images: 
  * main:3403caac05d30611ecf694b3d74e4ba4201c15fabbb1504112c81f35fb064975  # Different hash!
  * main:tag-4767
```

### Scenario 2: Fixed Tag (Demonstrates the Real Issue)

Even with a **fixed tag value**, the problem persists! This shows it's not about the tag changing, but about Pants' internal handling:

#### Run 1
```bash
$ RANDOM_TAG=tag pants package ::
...
Built docker images: 
  * base:289d383d39bc93699df66de3398a41ab7b236d6ee9b73ded0e825dfe8e3423e8  # Base hash is stable
  * base:tag
...
Built docker images: 
  * main:7c97563d83ffba8cafab21fbee29cedc6559a31dbe49ff19ff549719bd508b20  # Main hash
  * main:tag
```

#### Run 2 (same fixed tag)
```bash
$ RANDOM_TAG=tag pants package ::
...
Built docker images: 
  * base:289d383d39bc93699df66de3398a41ab7b236d6ee9b73ded0e825dfe8e3423e8  # Base hash UNCHANGED ✓
  * base:tag
...
Built docker images: 
  * main:efbf3dfcebd918d0a8c0905b3905cf532cffac4b174e96d19b664c788a746be6  # Main hash CHANGED! ✗
  * main:tag
```

#### Run 3 (still same fixed tag)
```bash
$ RANDOM_TAG=tag pants package ::
...
Built docker images: 
  * base:289d383d39bc93699df66de3398a41ab7b236d6ee9b73ded0e825dfe8e3423e8  # Base hash still UNCHANGED ✓
  * base:tag
...
Built docker images: 
  * main:032f12b36d315f0805db8126fe0bcb1b1810d22b7b45e8bc8c01518a39a2a4d4  # Main hash CHANGED AGAIN! ✗
  * main:tag
```

### Scenario 3: With Buildx Attestations Disabled (The Clearest Demonstration)

Docker buildx adds attestations that change on each build. By disabling them with `BUILDX_NO_DEFAULT_ATTESTATIONS=1`, we can see the pure tag-related issue:

#### Run 1 - Fixed tag with attestations disabled
```bash
$ BUILDX_NO_DEFAULT_ATTESTATIONS=1 RANDOM_TAG=tag pants package ::
...
Built docker images: 
  * base:cfb142dd3308bfe577086367f56122e0a2ee19393f9e66d440d7e2c54a18d684
  * base:tag
...
Built docker images: 
  * main:bbd45a84da6f035c4aea570c4f745aa37d98d811fef0f8c25689560f9a95e9b3
  * main:tag
```

#### Run 2 - Same tag, attestations still disabled
```bash
$ BUILDX_NO_DEFAULT_ATTESTATIONS=1 RANDOM_TAG=tag pants package ::
...
Built docker images: 
  * base:cfb142dd3308bfe577086367f56122e0a2ee19393f9e66d440d7e2c54a18d684  # Hash is stable! ✓
  * base:tag
...
Built docker images: 
  * main:bbd45a84da6f035c4aea570c4f745aa37d98d811fef0f8c25689560f9a95e9b3  # Hash is stable! ✓
  * main:tag
```

#### Run 3 - Different tag value
```bash
$ BUILDX_NO_DEFAULT_ATTESTATIONS=1 RANDOM_TAG=different-tag pants package ::
...
Built docker images: 
  * base:a1b2c3d4...  # Hash changes due to tag change ✗
  * base:different-tag
...
Built docker images: 
  * main:e5f6g7h8...  # Hash changes due to base's tag change ✗
  * main:different-tag
```

**This definitively proves:** The hash calculation incorrectly includes tag metadata. When tags change, hashes change, even though the actual Docker image content is identical.

## Why This Is a Problem

1. **Broken Caching:** Images are rebuilt unnecessarily because the hash changes based on tags
2. **CI/CD Inefficiency:** Builds take longer and use more resources
3. **Inconsistent Tags:** The `{pants.hash}` value changes even when image content is identical
4. **Circular Dependency:** The hash calculation includes tags, but `{pants.hash}` tag depends on the hash

> **Note on buildx attestations:** Docker buildx adds attestations that change on each build, which can mask the core issue. Use `BUILDX_NO_DEFAULT_ATTESTATIONS=1` to disable them and see the pure tag-related problem.

## Root Cause

The issue is in how Pants calculates the build context hash:

1. When building `main:image`, Pants first builds the upstream `base:image`
2. The `base:image` is packaged as a `BuiltPackage` which includes metadata containing all image tags
3. This metadata (including the dynamic `RANDOM_TAG`) becomes part of the package digest
4. The package digest is included in the build context snapshot for `main:image`
5. Since `RANDOM_TAG` changes on every build, the hash for `main:image` changes even if nothing else has changed

## Expected Behavior

The hash should remain stable when:
- No source files have changed
- No dependencies have changed  
- No build configuration has changed

The `image_tags` field should not affect the hash calculation because tags are metadata about how to publish the image, not part of the image content itself.

## Real-World Impact

This bug affects any project using:
- Multi-stage Docker builds with Pants
- Dynamic tags (timestamps, git commits, branch names)
- CI/CD pipelines that rely on caching

Common patterns that trigger this bug:
```python
image_tags=[
    "{pants.hash}",
    env("BRANCH_NAME"),
    env("COMMIT_SHA"),
    f"latest-{env('ENVIRONMENT')}",
    f"{env('VERSION')}-{env('BUILD_NUMBER')}",
]
```

## Environment

- **Pants Version:** 2.26.1 (likely affects other versions)
- **Docker:** Standard Docker daemon
- **Platform:** linux/amd64

## Suggested Fix

The `image_tags` metadata should be excluded from the digest calculation used for determining the build context hash, as tags are publishing metadata rather than build inputs.