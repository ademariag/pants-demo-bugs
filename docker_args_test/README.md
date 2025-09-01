# Docker Build Args with Target References - Issue #22602

## Bug Report

Pants does not properly support target references when passed as `extra_build_args` in Docker image targets. When users try to pass target addresses as build arguments, Pants fails to resolve them to the actual built Docker image tags, causing build failures.

## Reproducing the Issue

This directory demonstrates the problem with a concrete example. The following configuration currently fails:

**BUILD file:**
```python
docker_image(
    name="base",
    source="Dockerfile.base",  # FROM ubuntu
)

docker_image(
    name="base2", 
    source="Dockerfile.base2",  # FROM alpine
)

docker_image(
    name="main",
    source="Dockerfile",
    extra_build_args=[
        "BASE_IMAGE=:base"  # Target reference - would not resolve properly
    ]
)
```

**Dockerfile:**
```dockerfile
ARG BASE_IMAGE=:base
FROM ${BASE_IMAGE}
RUN if command -v apt-get >/dev/null 2>&1; then apt-get update && apt-get install -y python3 python3-pip; elif command -v apk >/dev/null 2>&1; then apk add --no-cache python3 py3-pip; fi
```

The `BASE_IMAGE=:base` argument is not being resolved to the actual Docker image tag of the built `:base` target, causing the `FROM ${BASE_IMAGE}` instruction to fail.

## Expected Behavior

The target reference `:base` in the `extra_build_args` should be resolved to the actual Docker image tag produced by building the `:base` target (e.g., `sha256:abc123...` or a tagged name), allowing the Dockerfile's `FROM ${BASE_IMAGE}` instruction to work correctly.

This would enable powerful Docker image composition patterns where one Docker image can reference another Pants-built Docker image as its base image through build arguments.

## Current Error

When running:

```bash
pants package docker_args_test:main
```

The build fails because `${BASE_IMAGE}` resolves to the literal string `:base` instead of the actual built Docker image identifier, causing Docker's `FROM` instruction to fail with an invalid image reference.

## Impact

This limitation prevents users from:
- Creating hierarchical Docker image dependencies within Pants
- Dynamically selecting base images through build arguments  
- Composing complex Docker build pipelines where images depend on other Pants-built images

## Workaround

Currently, users must hardcode image tags or use external tooling to pass the correct image references, defeating the purpose of Pants' dependency management for Docker images.