# ForgeFlow

A simple CI automation tool for building and testing software.

## Overview

ForgeFlow is a CI system written in Go, designed as a project to explore and implement the core concepts behind continuous integration systems.

The project focuses on building a CI system from the ground up, including workflow configuration, job execution, dependency management, parallel execution, logging, and reporting.

## Goals

- Define CI workflows through configuration
- Execute independent jobs in parallel
- Resolve dependencies between jobs
- Capture and expose job output
- Provide clear job and workflow status
- Run reliably in automated environments
- Keep the system modular and extensible

## Scope

### In scope

- Workflow definition
- Build and test stages
- Job scheduling
- Dependency resolution
- Process execution
- Logging
- Failure handling
- CLI
- Configuration

### Out of scope

- Distributed execution
- Hosted CI infrastructure
- Repository hosting

## Status

Early development.

The architecture and design are currently being defined before implementation begins.

## Technology

- Go
- Git
