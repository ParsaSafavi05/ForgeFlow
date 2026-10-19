# ForgeFlow Architecture

This document explains how the code is organized and how the pieces talk to each other.

## Folder Layout

```
cmd/forgeflow/        CLI entrypoint (main.go)
internal/config/      read YAML, validate, produce Pipeline
internal/graph/       build the DAG, detect cycles, topological sort
internal/scheduler/   the run loop: decides what runs and when
internal/runner/      Runner interface + local (os/exec) implementation
internal/docker/      (later) DockerRunner, image build / tag / push
internal/logs/        captures and stores job output
internal/report/      prints the final summary
```

`internal/` is a Go convention: code inside it can only be imported by this project, which keeps our internals private.

## Core Types

These are a starting point and will change as the config format is decided.

```go
type Pipeline struct {
    Name string
    Jobs map[string]Job
}

type Job struct {
    Name    string
    Run     string            // command to execute
    Needs   []string          // names of jobs that must succeed first
    Env     map[string]string
    Timeout time.Duration
    // later: Image string  (run inside this container image)
}

type Status string // pending, running, success, failed, skipped, cancelled

type Result struct {
    JobName  string
    Status   Status
    ExitCode int
    Duration time.Duration
    Err      error  // e.g. "could not start process"
    // later: ImageDigest string (for image build/push jobs)
}
```

## The Runner Interface

The most important design decision: the scheduler does not know how jobs are executed.

```go
type Runner interface {
    Run(ctx context.Context, job Job, out io.Writer) Result
}
```

- `LocalRunner` uses `os/exec` to run a command on the host.
- `DockerRunner` (later) runs the same command inside a container.
- Tests use a fake that returns instant, scripted results.

This lets us test scheduling logic (dependencies, skipping, parallelism) without running real commands, and it means adding Docker does not require changing the scheduler.

Picking a runner per job is the job of a small dispatcher: if the job asks for a container image, use `DockerRunner`, otherwise use `LocalRunner`.

## How the Scheduler Works

In simple terms, the scheduler is a loop:

```
1. Find all jobs that are ready (pending + all dependencies succeeded).
2. Start them, but never more than the concurrency limit.
3. Wait for any running job to finish.
4. Record its result.
   - If it failed: mark everything that depends on it as skipped.
5. Go back to step 1.
6. Stop when nothing is pending and nothing is running.
```

### Why it's safe

Only **one goroutine** (the scheduler loop) reads and writes job states. Worker goroutines run the jobs and send a `Result` back through a channel. Because only one goroutine touches the state, we don't need locks, and we avoid a whole class of race-condition bugs.

```
            ┌───────────────┐
            │   Scheduler   │  (owns all state)
            └──┬─────────▲──┘
   start job   │         │  results (channel)
               ▼         │
        ┌────────┐  ┌────────┐
        │Worker 1│  │Worker 2│ ...   (each calls Runner.Run)
        └────────┘  └────────┘
```

## How the Local Runner Executes a Process

Details that are easy to get wrong:

- **Use `exec.CommandContext`** so timeouts and cancellation actually stop the process.
- **Use a process group** (on Unix, `Setpgid`) and kill the whole group. Otherwise, when you kill `make` or `npm`, the programs they started keep running in the background.
- **Capture stdout and stderr together** in one stream, in order, so logs read naturally.
- **Tell the difference** between "the process could not start" (a config or environment problem) and "the process ran and exited non-zero" (a real job failure).

## Docker in the Architecture

Everything about Docker lives in `internal/docker/` and in docker.md. The short version:

- `DockerRunner` implements the same `Runner` interface, so the scheduler is untouched.
- Image build, tag and push are small functions in the same package, used by special job kinds.
- Docker is first accessed by calling the `docker` command-line tool, the same way `LocalRunner` calls any other command. Using the Docker Engine API directly can come later if it's needed.
- Cancellation is different for containers: killing the `docker` CLI process does **not** stop the container, so `DockerRunner` must give each container a name and call `docker kill` on cancel.

## Logs

Each job writes its output to its own log file, for example `.forgeflow/logs/<run-id>/<job>.log`. Output can also be shown live in the terminal with a prefix:

```
[build] compiling...
[lint ] checking 42 files...
```

Logs must never contain secrets. Anything marked as secret (such as a registry password) is masked before it is written.

## Reporting With Events

The scheduler emits small events as things happen:

```
JobStarted, JobFinished, JobSkipped, PipelineFinished
```

The reporter listens to these events. This means the same system can support:

- a final text summary (first version)
- live progress in the terminal (later)
- JSON output for other tools (later)

...without changing the scheduler.

## Cancellation (Ctrl+C)

The CLI creates a `context.Context` at start. When the user presses Ctrl+C:

1. The context is cancelled.
2. The scheduler stops starting new jobs.
3. Running jobs receive the cancellation. Local processes have their process groups killed; containers are stopped with `docker kill`.
4. Jobs are marked `cancelled` and the report is still printed.

## Testing Strategy

| Layer        | How to test                                                                                                                             |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| config       | table tests with valid and invalid YAML                                                                                                 |
| graph        | cycles, missing deps, diamond shapes, topological order                                                                                 |
| scheduler    | fake `Runner`: check order, parallelism limit, skip-on-failure                                                                          |
| local runner | small real commands (`echo`, `exit 1`, `sleep`)                                                                                         |
| docker       | unit tests on the docker commands that get built; real Docker tests marked as integration tests and skipped when Docker isn't available |
| end to end   | run the CLI on sample `forgeflow.yaml` files                                                                                            |