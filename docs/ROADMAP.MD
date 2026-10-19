# ForgeFlow Roadmap

These are the steps which should be taken carefully in order to develop ForgeFlow step-by-step. Each milestone will be tested before moving onto the next

## Milestone 1: Config (provisional)

- [ ] Define `Pipeline` and `Job` structs
- [ ] Parse `forgeflow.yaml`
- [ ] Validate: missing fields, duplicate names, unknown dependencies
- [ ] Unit tests with valid and invalid files
- [ ] Decide the open questions listed in [design.md](DESIGN.md)

## Milestone 2: Graph

- [ ] Build the dependency graph
- [ ] Detect cycles
- [ ] Topological sort (Kahn's algorithm)
- [ ] Tests: simple chain, diamond shape, cycle, independent jobs

## Milestone 3: Runner

- [ ] `Runner` interface
- [ ] `LocalRunner` using `os/exec`: run one command, return exit code and duration
- [ ] Capture stdout and stderr into a log
- [ ] Distinguish "failed to start" from "non-zero exit"

## Milestone 4: Sequential Scheduler

- [ ] Run jobs one at a time in dependency order
- [ ] Mark dependents as `skipped` when a job fails
- [ ] Tests using a fake `Runner`

## Milestone 5: Parallel Scheduler

- [ ] Run all ready jobs at once
- [ ] Concurrency limit (`--jobs N`)
- [ ] Single goroutine owns state; workers send results over a channel
- [ ] Run tests with `go test -race`

## Milestone 6: Robustness

- [ ] Per-job timeouts
- [ ] Ctrl+C handling with `context` cancellation
- [ ] Kill whole process group
- [ ] `--fail-fast`

## Milestone 7: Reporting and CLI

- [ ] Final summary (status, duration per job)
- [ ] Per-job log files
- [ ] Live output with `[job]` prefixes
- [ ] `forgeflow run` and `forgeflow validate`
- [ ] Meaningful exit codes

> The core CI tool is complete here. The milestones below add Docker. See docker.md.

## Milestone 8: Image Build and Tag

- [ ] Check that Docker is available before running (clear error if not)
- [ ] Docker wrapper interface in `internal/docker/` with a fake for tests
- [ ] `docker build` job from a Dockerfile
- [ ] Tag generation from git (short commit, branch, version tag)
- [ ] Show image name and tags in the report
- [ ] Unit tests for command building and tag logic

## Milestone 9: Image Push

- [ ] Registry login using credentials from environment variables
- [ ] Password passed via stdin, secrets masked in logs and reports
- [ ] `docker push` job, ordered after build and tests by the DAG
- [ ] Record and report the image digest
- [ ] Safe default: no accidental push when running locally (dry-run / `--no-push`)
- [ ] Integration test against a local test registry

## Milestone 10: Jobs Inside Containers

- [ ] `DockerRunner` implementing the `Runner` interface
- [ ] Runner dispatcher: container if the job names an image, otherwise local
- [ ] Mount the project folder, set working directory and environment
- [ ] Named containers; stop them with `docker kill` on timeout or Ctrl+C
- [ ] Pull each image once per run
- [ ] Handle file ownership problems
- [ ] Integration tests (skipped when Docker isn't available)

## Later Ideas

- JSON output
- Job `steps`
- Matrix builds (same job with different parameters)
- Caching between runs (including Docker layer caching)
- Conditional jobs (`if:`)
- Retry on failure
- Docker Engine API instead of the CLI
- Multi-platform images