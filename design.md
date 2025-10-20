# Job Runner Service

This document describes a solution for a remote job execution service to run arbitrary linux processes.

## Design Principles

This design aims to be modular, performant, and reliable. It implements the core components according to best practices, with the minimum required functionality, while noting areas for future development.

It leverages the strengths of Go for concurrent workflows and gRPC for performant, secure communication.


## Overview

The system comprises of 3 core components.
- A job manager library to abstract job execution, persistence, and queries.
- A CLI app to start, stop, monitor and list jobs.
- A Server that responds to each client action by delegating to the job manager library.

The job manager handles job actions, and persists each job with specific states such as `pending`, `running`, or `completed`, as well as their outputs for future queries.

There is no restriction on the commands to be executed and they can target any program available on the machine hosting the gRPC service.

The following diagram illustrates the core modules and how they relate to each other.

![Job Runner Diagram](JobRunnerDiagram.jpg)

## Authentication and Authorization

The system uses mTLS authentication to create a secure communication channel between CLI clients and the gRPC server.

Ther server uses the identity information contained in the client certificate to determine the client's role based on their CN name; either `user` or `admin`.

### Role differences
- Users can list, monitor and stop jobs that they submitted.
- Admins can list, monitor and stop jobs submitted by _any_ user.

## Security Vulnerabilities

Since the app explicitly enables the execution of any program on the server, vulnerabilities include
- Downloading and running malware using `curl`
- Reading sensitive data or config files with `cat`
- Destroying files using `mv path/to/file /dev/null` or `rm`
- Mutating config values using `sed`
- Killing running processes for disruption
- Launching processes that exhaust system resources

In short, a user can wreak catastrophic havoc on the system and organization.

If commands can be executed in a sandboxed environment without direct access to the server host, some of these risks can be mitigated.


## CLI

The `jobr` CLI app is a user-friendly interface for job management.

`-h` or `--help` - Provides CLI usage information.


For mTLS authentication, the client must set two environment variables:
- `SSL_CERT` - Path to public certificate containing identity information and public key.
- `SSL_KEY` - Path to private key file to generate a signature that the server verifies.


### Start

```sh
start "<command with args>"
```

Submits the given shell command to the server and tails the job output, until the user presses ctrl+c. The job continues processing even if the client is no longer tailing the output.

Example
```sh
$ jobr start "echo Hello!"
Hello!
```



### Stop

```sh
stop <jobId>
```

Stops the job denoted by the given `jobId`. The `jobId` is a 5 character GUID assigned to each submitted job by the job manager which can be queried using the `ls` command.

Example
```sh
$ jobr stop 7766b
stop succeeded
```

### List jobs

```sh
ls
```

Lists all jobs submitted by the user. If the user is an admin, then lists jobs submitted by all users.

Example
```sh
$ jobr ls
JOB ID     COMMAND       STATUS     DURATION
7766b      "echo Hello!" completed  1ms
112ab      "echo test"   completed  1ms
```


### Monitor

```sh
monitor <jobId>
```

Queries the job's output from the beginning and tails it until the user presses ctrl+c.

Example
```sh
$ jobr monitor 112ab
test
```


## API Server

The gRPC Server implements handlers for all supported job actions listed in the `.proto` file. For certain rpc methods, the responses can change based on the client's role derived from their certificate identity.

## Job Manager API Libary

The job manager library implements the functionality to start, stop, persist and monitor job executions on the server.

It is responsible for
- Accepting new jobs for concurrent job execution using a worker pool.
- Persisting job metadata and outputs for each submitted job.
  - Creating a 5 character GUID for a job
  - Calculating job duration by using the job start time from the present.
- Updating the job status appropriately
  - `pending` for jobs accepted but not started
  - `running` for jobs executing on the server
  - `completed` for jobs that finished execution successfully
  - `failed` for jobs that finished execution with a nonzero exit code, or timed out
  - `stopped` for jobs that were explicitly stopped by a user or admin
- Tailing job output independent of job execution.
- Gracefully handling stopping a job by terminating the process as well as the observation of its output/status.

## Tests

Automated tests for the following behaviors will be implemented:
- Server should return jobs scoped to a user when queried by a user, and all jobs when queried by an admin.
- Server should reject requests from clients with an invalid certificate (mTLS auth middleware test).
