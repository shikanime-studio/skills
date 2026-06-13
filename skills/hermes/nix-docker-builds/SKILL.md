---
name: nix-docker-builds
description:
  "Build Docker images with Nix: pure Nix vs hybrid builder patterns, rootfs
  packaging pitfalls, and when dockerTools output is directly usable."
---

# Nix + Docker Builds

Skill for building container images with Nix + Dockerfiles while keeping builds
reproducible and Dockerfiles minimal.

## When to use the hybrid pattern

Trigger: user asks for a `Dockerfile`, pure Nix is blocked, or they want
`dockertools` semantics inside a normal Docker build. The goal is to let Nix
produce the image rootfs, then stage it in Docker as cleanly as possible.

## Preferred shape

Two options:

1. Pure Nix: `nix build` produces a Docker image tarball →
   `docker load -i result`.
2. Hybrid: `Dockerfile` + Nix builder stage, where the builder runs `nix build`,
   copies `/nix/store` closure plus the app symlink into `scratch`, then just
   copies/paths the produced artifact(s).

## Rootfs packaging gotchas

- `nix build` often resolves `result` to a symlink in `/nix/store`. Always
  resolve it with `readlink -f result` if you need the actual path.
- When using `dockerTools.buildLayeredImage`/`buildImage` in a flake, the
  package output is a Docker image tarball, not a filesystem tree. That `tar.gz`
  is not `COPY`-able into `scratch`.
- If the flake exposes a plain binary package (via `buildGoModule`, etc.), the
  result is a directory tree that Docker `COPY` understands. Prefer exposing
  both when the user wants a Dockerfile optional path.

## Template

Use `references/dockerfile-nix-builder.template` as the starter. It follows
Mitchell’s pattern with experimental features enabled and `readlink -f result`
resolution.
