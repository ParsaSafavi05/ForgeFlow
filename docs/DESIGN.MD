# ForgeFlow Design

ForgeFlow is a small CI tool written in Go. You describe your build in a YAML file, and ForgeFlow runs it for you in the right order, as fast as possible, and with a clear report at the end. It can also run jobs inside Docker containers, and build, tag and push Docker images of your application.

## The Big Idea

A CI pipeline is a set of **jobs** (like "build", "test", "lint"). Some jobs need others to finish first. Some don't care about each other and can run at the same time.

```
YAML file -> Pipeline -> check it -> schedule jobs -> run them -> report
```

The job model in this project is implemented as a **DAG** (Directed Acyclic Graph). In simple terms, each job only starts once the jobs it depends on have finished, and jobs that don't depend on each other can run at the same time. "Acyclic" means there are no loops: A can't wait for B if B is waiting for A.

```
build ──> test ──┐
                 ├──> package
lint ────────────┘
```

Here `build` and `lint` can start at the same time. `test` waits for `build`. `package` waits for both `test` and `lint`.

## The Components

ForgeFlow has 6 core components. Each one has one job and a clear input and output.

|#|Component|Input|Output|
|---|---|---|---|
|1|**Config reader**|`forgeflow.yaml`|a `Pipeline` struct|
|2|**Validator / graph builder**|`Pipeline`|a valid DAG, or an error (cycle, unknown dependency, ...)|
|3|**Scheduler**|the DAG|decisions: which job to start next|
|4|**Runner**|one job|exit code, duration, logs|
|5|**Log store**|job output|log files / terminal output|
|6|**Reporter**|all job results|final summary|

Why split the graph part into two (validator + scheduler)?

- The **validator** runs once, before anything starts. It catches mistakes early ("job `test` needs `biuld`, which doesn't exist").
- The **scheduler** runs the whole time while jobs execute, because which jobs are ready changes every time a job finishes or fails.

### Docker support (added in later milestones)

Docker is not a new stage in the pipeline. It plugs into the existing components:

|Feature|Where it plugs in|
|---|---|
|**Run a job inside a container**|A second Runner implementation (`DockerRunner`). The scheduler doesn't change.|
|**Build an image**|A kind of job that turns into `docker build`|
|**Tag an image**|Part of the build step, using tags like the git commit|
|**Push an image to a registry**|A kind of job that turns into `docker login` + `docker push`|

Full details are in [docker.md](DOCKER.md).

## Job States

Every job is always in exactly one of these states:

```
pending -> running -> success
                   -> failed
pending -> skipped     (a dependency failed)
any     -> cancelled   (user pressed Ctrl+C, timeout, or fail-fast)
```

A job is **ready** when it is `pending` and all of its dependencies are `success`.

## Failure Rules

- If a job fails, every job that depends on it (directly or indirectly) becomes `skipped`.
- If a step fails, the remaining steps of that job are not run and the job is marked `failed`.
- Jobs that don't depend on the failed job keep running.
- Optional `--fail-fast` flag: cancel everything as soon as one job fails.
- The pipeline's final status is `success` only if every job succeeded.
- Exit code of `forgeflow`: `0` if the pipeline succeeded, non-zero otherwise (so other tools can use it).
- This also protects pushes: an image push job that needs `test` will never run if `test` fails.

## Design Principles

1. **Simple first.** Get a correct sequential version working before adding parallelism, and local execution before Docker.
2. **Testable.** The scheduler never launches real processes directly. It talks to a `Runner` interface, so tests can use a fake runner.
3. **One owner of state.** Only the scheduler changes job states. Workers just report results back.
4. **Fail loudly and early.** Bad config is rejected before any job runs.
5. **Extensible.** Reporter, runner, and config format are separate pieces that can be swapped or extended. Docker is added as one more runner, not by rewriting the core.
6. **Safe with secrets.** Registry passwords and tokens never appear in logs or reports.

## Decisions

These choices shape the config format. Anything not listed here is still undecided.

- **Jobs have steps.** A job has a list of `steps`, and each step is one command. Steps run in order, and the first failing step fails the job and skips the remaining steps. A one-command shorthand can be added later without breaking anything.
- **Dependencies use `needs`.** It lists the jobs that must succeed first.
- **Unknown fields are rejected.** A typo like `imge:` is an error, not silently ignored. This also means new fields can be added later without breaking old files.
- **Job names** may contain only letters, digits, `-` and `_`. They end up in log file names and container names.
- **Defaults:** no timeout, and the working directory is the project root.
- **Container execution (later):** an optional `image` field on a job. If absent, the job runs locally. This is not part of the first parser. See [docker.md](docker.md).
- **Image build and push (later):** until Milestones 8 and 9, users write `docker build` and `docker push` as normal steps. Dedicated job kinds are added later as optional new keys.
- **Secrets:** the pipeline file names the environment variable that holds a secret, never the value. ForgeFlow reads it at run time, masks it in logs, and fails early if it is not set.


## Out of Scope

- Distributed execution (running jobs on many machines)
- Hosted CI infrastructure
- Repository hosting
- Container orchestration (Kubernetes and similar)