# ForgeFlow Docker Support

Docker support is planned for **later milestones**, after the core pipeline works. It is written down now so the early design leaves room for it.

There are two separate features:

1. **Run jobs inside containers**, so each job gets a clean, reproducible environment.
2. **Build, tag and push images** of your application to a registry.

The configuration format is not decided yet. YAML snippets in this document are **illustrations only**.

## Feature 1: Running Jobs in Containers

### What it means

Normally a job's command runs directly on your machine. With Docker, the same command runs inside a container started from an image you choose, for example `golang:1.23`. The project folder is made available inside the container, so the command can read and build your code.

Why this is useful:

- Everyone gets the same tools and versions, regardless of what's installed on the host.
- Jobs can't mess up the host machine.
- Different jobs can use different environments.

### How it fits the design

`DockerRunner` is a second implementation of the `Runner` interface (see [architecture.md](ARCHITECTURE.md)). The scheduler doesn't know or care whether a job ran in a process or a container.

### What DockerRunner does for one job

```
1. Make sure the image exists (pull it if needed).
2. Start a container with:
   - a unique name, e.g. forgeflow-<run-id>-<job>
   - the project folder mounted in
   - the working directory set
   - the job's environment variables
   - automatic removal when it finishes
3. Stream the container's output into the job's log.
4. Wait for it to exit and use its exit code as the job's exit code.
5. On timeout or Ctrl+C: stop the container by name (docker kill).
```

### Things to watch out for

|Problem|Why it matters|Plan|
|---|---|---|
|Cancellation|Killing the `docker` CLI process does not stop the container|Name containers and `docker kill` them|
|File ownership|Files created inside a container may be owned by root on the host|Run as the host user, or document the limitation|
|Docker not available|Daemon not running, or not installed|Check once before the run and fail with a clear message, as a "failed to start" error rather than a job failure|
|Slow image pulls|Parallel jobs may pull the same image|Pull each image once per run|
|Leftover containers|Crashes could leave containers running|Name them with the run ID so they can be cleaned up|

### Illustration

```yaml
jobs:
  test:
    needs: [build]
    image: golang:1.23      # run inside this container
    run: go test ./...
```

## Feature 2: Build, Tag and Push Images

### What it means

After your code builds and the tests pass, ForgeFlow can:

1. **Build** a Docker image from your `Dockerfile`
2. **Tag** it with meaningful names
3. **Push** it to a registry (Docker Hub, GitHub Container Registry, a private registry, ...)

### How it fits the design

These are ordinary jobs in the DAG. Pushing depends on building, and building can depend on tests:

```
build ──> test ──> image-build ──> image-push
```

Because of the failure rules in design.md, if `test` fails, `image-build` and `image-push` are skipped automatically. A broken version never gets pushed.

Build and push run on the host through the `docker` CLI (they talk to the Docker daemon directly), so they use the local runner rather than running inside a container.

### Tagging

A tag is a label on an image, like `myapp:1.4.0`. A good CI tool tags with information that identifies the build. Candidates:

|Tag|Example|When it's useful|
|---|---|---|
|Short git commit|`myapp:3f2a9c1`|Always unique, traces back to the exact code|
|Branch name|`myapp:main`|"latest build of this branch"|
|Version from a git tag|`myapp:v1.4.0`|Releases|
|`latest`|`myapp:latest`|Only for releases or the main branch, never by default|

Git is used to find the commit, branch and tag, which is why Git is already in the technology list.

### Pushing and registry login

Pushing needs credentials. Rules:

- Credentials come from **environment variables**, never from the pipeline file itself.
- Login uses the password through standard input, so it never appears in a command line or process list.
- Secrets are **masked in logs and reports**.
- Push only runs when explicitly part of the pipeline. Running `forgeflow run` locally should not push by accident (for example, via a `--no-push` or dry-run option).

### What gets reported

For image jobs the reporter should show:

- the full image name and tags
- the **digest** (the unique ID of the pushed image, like `sha256:ab12...`)
- whether the push succeeded

### Illustration

```yaml
jobs:
  image:
    needs: [test]
    docker_build:
      dockerfile: Dockerfile
      image: ghcr.io/me/myapp
      tags: [commit, branch]

  publish:
    needs: [image]
    docker_push:
      image: ghcr.io/me/myapp
```

## Implementation Approach

1. **Use the `docker` command-line tool first.** It's simple, needs no extra Go dependencies, and works the same way as the local runner. The Docker Engine API (Go SDK) can replace it later if needed.
2. **Hide Docker behind a small interface** in `internal/docker/`, so tests can use a fake instead of requiring Docker.
3. **Build the command lists as pure functions** (for example, "given this job, what `docker run` arguments do we need?"). These are easy to unit test without Docker installed.

## Testing

- **Unit tests:** command construction, tag generation, secret masking. No Docker needed.
- **Integration tests:** real `docker run`, `build` and `push` against a local test registry. Marked with a build tag (`//go:build integration`) and skipped when Docker isn't available.

## Security Notes

- Access to the Docker daemon is powerful. Document that ForgeFlow needs it and what that means for shared machines.
- Never print credentials. Mask them in logs, errors and reports.
- Don't mount more of the host into containers than the project folder.

## Out of Scope (for now)

- Building images without a Docker daemon (Kaniko, Buildah)
- Multi-platform builds (arm64 + amd64)
- Image scanning and signing
- Container orchestration such as Kubernetes