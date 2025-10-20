# Job Runner Service

This document describes a solution for a remote job execution service to run arbitrary linux processes.

## Design Principles

This design aims to be modular, performant, and reliable. Implementation of the core components will strive to attain the minimum required functionality with expressive APIs that work well with each other.

Opportunities for improvements, or abbreviations of certain features will be indicated in comments.

The implementation strives to showcase the strengths of Go for managing concurrent workflows and gRPC for performant, secure communication.


## Overview

The system comprises of 3 core components.
- A CLI app to start, stop, monitor and list jobs.
- A Server that delegates actions to a job manager library and returns results to the client.
- A job manager library to abstract the implementation details of job execution, persistence, and queries.

There is no restriction on the commands to be executed and they can target any program available on the machine hosting the gRPC server.

A job's output can be text or raw binary data and is streamed to the client when they start or monitor a job.

The following diagram illustrates the commands, core modules and how they relate to each other.

![Job Runner Diagram](JobRunnerDiagram.jpg)

## Authentication and Authorization

The system uses mTLS authentication to create a secure communication channel between CLI clients and the gRPC server.

The server uses the CN name contained in the client certificate to determine the client's role; either `user` or `admin`.

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

If commands can be executed in a sandboxed environment with bounded resources, and without direct access to the server host, some of these risks can be mitigated.


## CLI

The `jobr` CLI app is a user-friendly interface for remote job execution and management.

`-h` or `--help` - Provides CLI usage information.

For mTLS authentication, the client must set environment variables:
- `CLIENT_CERT_FILE` - Path to public certificate containing identity information and public key.
- `CLIENT_KEY_FILE` - Path to private key file to generate a signature that the server verifies.
- `CA_FILE` - Path to CA cert to verify server certificate.

### Commands 

#### Start

`start "<command with args>"`

Submits the given shell command to the server and tails the job output, until the user presses ctrl+c. 
The assigned job id is printed first so that the user can use it for other commands.
The job continues processing even if the client is no longer tailing the output.

Example
```sh
$ jobr start "echo Hello!"
Job id: 7766b
Hello!
```


#### List jobs

`ls`

Lists all jobs submitted by the user. If the user is an admin, then lists jobs submitted by all users.

Example
```sh
$ jobr ls
JOB ID     COMMAND       STATUS     DURATION START TIME
7766b      "echo Hello!" completed  1ms      2025-10-20T22:10:04.191Z
112ab      "echo test"   completed  1ms      2025-10-20T22:11:00.191Z
```


#### Stop

`stop <jobId>`

Stops the job denoted by the given `jobId`. The `jobId` is a 5 character GUID assigned to each submitted job by the job manager which can be queried using the `ls` command.
A user can stop jobs they have started, while an admin can stop any job.
An empty response indicates success, while an error with a status code provides information for why the job couldn't be stopped (NOT_FOUND, UNAUTHORIZED).

Example
```sh
$ jobr stop 7766b
stop succeeded
```


#### Monitor

`monitor <jobId>`

Queries the job's output from the beginning and tails it until the user presses ctrl+c.
A user can monitor jobs they have started, while an admin can monitor any job.

Example
```sh
$ jobr monitor 112ab
test
```


## GRPC Server

The gRPC Server implements handlers for all supported job actions listed in the `.proto` file. For certain rpc methods, the responses can change based on the client's role derived from their certificate identity.

The server is responsible for
- Implementing all RPC methods defined in the `.proto` file with appropriate responses and status codes.
- Authentication using mTLS with client and server certificates and authorization using CN name derived from the client certificate.
- Wrapping the job manager library and graceful shutdown of all goroutines when exiting.


For mTLS authentication, the server env must set environment variables:
- `SERVER_CERT_FILE` - Path to public certificate containing identity information and public key.
- `SERVER_KEY_FILE` - Path to private key file to generate a signature that the client verifies.
- `CA_FILE` - Path to CA cert to verify client certificate.

## Job Manager API Libary

The job manager library implements the functionality to start, stop, persist and monitor job executions on the server.

Note that all data is persisted in-memory and will be lost when the GRPC Server restarts.

It is responsible for
- Accepting new jobs for concurrent job execution using a worker pool.
- Persisting job metadata and outputs for each submitted job.
  - Job ID is a 5 character GUI
  - Job duration based on how much time was spent in the running state.
- Updating the job status appropriately
  - `pending` for jobs accepted but not started
  - `running` for jobs executing on the server
  - `completed` for jobs that finished execution successfully
  - `failed` for jobs that finished execution with a nonzero exit code, or timed out
  - `stopped` for jobs that were explicitly stopped by a user or admin
- Tailing job output independent of job execution.
- Gracefully stopping a job by first terminating the launched OS process and it's calling goroutine.

## Tests

Automated tests for the following behaviors will be implemented:
- Server should return jobs scoped to a user when queried by a user, and all jobs when queried by an admin (authorization scope test)
- Server should reject requests from clients with an invalid certificate (mTLS auth middleware test).
- Job manager should allow executing, and monitoring jobs concurrently without data races.