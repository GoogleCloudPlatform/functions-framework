# Functions Framework

The Google Cloud Function Frameworks are open source libraries for writing portable Google Cloud functions -- brought to you by the Google Cloud Functions team.

The Functions Framework lets you write lightweight functions that run in many
different environments, including:

*   [Google Cloud Functions](https://cloud.google.com/functions/)
*   Your local development machine
*   [Cloud Run and Cloud Run on GKE](https://cloud.google.com/run/)
*   [Knative](https://github.com/knative/)-based environments

These functions allow you to easily handle **HTTP requests** or **[CloudEvents](https://cloudevents.io/)** in a Functions as a Service (FaaS) programming style.

## Video

<a href="https://www.youtube.com/watch?v=qQiqo1zZJmI">
<img width="496" alt="Watch a video about the Functions Framework" src="https://user-images.githubusercontent.com/744973/71794367-bdafaa80-3073-11ea-8728-61d24942bc56.png">
</a>

## Languages

The Functions Framework is implemented for these runtimes:

- [Node.js](https://github.com/GoogleCloudPlatform/functions-framework-nodejs)
- [Go](https://github.com/GoogleCloudPlatform/functions-framework-go)
- [Java](https://github.com/GoogleCloudPlatform/functions-framework-java)
- [PHP](https://github.com/GoogleCloudPlatform/functions-framework-php)
- [Python](https://github.com/GoogleCloudPlatform/functions-framework-python)
- [Ruby](https://github.com/GoogleCloudPlatform/functions-framework-ruby)
- [.NET](https://github.com/GoogleCloudPlatform/functions-framework-dotnet)

## Specification Summary

A Functions Framework instantiates web server and invokes function code in response to an HTTP request (`http`) or CloudEvent request (`cloudevent`) depending on the function's signature type. A Functions Framework may optionally support functions with signature type `event` for legacy-style events.

The Functions Framework library must be configurable via environment variables and may be configurable via command-line flags:

Command-line flag         | Environment variable      | Description
------------------------- | ------------------------- | -----------
`--port`                    | `PORT`                    | The port on which the Functions Framework listens for requests. Default: `8080`
`--target`         | `FUNCTION_TARGET`         | The name of the exported function to be invoked in response to requests. Default: `function`
`--signature-type` | `FUNCTION_SIGNATURE_TYPE` | The signature used when writing your function. Controls unmarshalling rules and determines which arguments are used to invoke your function. Default: `http`; accepted values: `http` or `cloudevent`

> Note: `SIGNATURE_TYPE: event` supports legacy, non-CloudEvent events. Support for these event formats are not required for Function Frameworks.

-----

# Functions Framework Contract

> Note: This section is useful for Function Framework builders. Developers should view individual
> function framework repos for how to use a specific framework.

This contract builds upon the baseline compliance of the existing Cloud Run contract (e.g. the [Knative Runtime Contract](https://github.com/knative/serving/blob/master/docs/runtime-contract.md)), which itself is built on OCI.

## Goal

Functions Frameworks aim to minimize the amount of boilerplate and configuration required to create a runnable stateless container. The overall goal is to maximize productivity and free the developer from repetitive development tasks that do not directly contribute to solving customer problems.

## Components

A Functions Framework consists of **two parts**:
- A package that instantiates a web server and invokes function code in response to an HTTP request. This package may include additional functionality to further minimize boilerplate.
- A script or tool that converts a source code transform of function code into app code ("the function-to-app converter") 

## Runtime and Lifecycle

### Statefulness

Functions are generally deployed to stateless compute environments. In such an environment, the
container or other environment that is running the function may be instantiated from scratch, paused, started or stopped
based on inbound request volume.

The framework should be able to gracefully handle these dynamics, for example, by minimizing container and framework startup times. The framework should be long-lived and able to handle repeated invocations of the developer's function.

### Lifecycle

The framework must load the function located and defined by the `FUNCTION_TARGET` environment variable on startup and create a local web server that listens for inbound HTTP requests. As stated in the Knative serving contract ([Process](https://github.com/knative/serving/blob/master/docs/runtime-contract.md#process)), the web server must listen for ingress requests on the port defined by the `PORT` environment variable.

When a caller sends an HTTP request to the web server, the framework must take unmarshalling steps according to the developer's specified function signature type. The function must then be invoked by passing appropriate arguments conforming to the developer's specified function signature type.

When the function signals that it has completed executing (i.e., "completion signalling"), the framework must respond to the caller. Depending on the developer's function signature type, the framework may first need to marshall objects returned by the developer's function into an HTTP response.

No work is done after a response is sent.

For performance, efficiency and correctness reasons, the framework must be able to handle multiple concurrent invocations of the developer's function.

When the framework and function are deployed as a container to a Knative environment, additional [Lifecycle considerations](https://github.com/knative/serving/blob/master/docs/runtime-contract.md#lifecycle) apply.  As a general rule of thumb, frameworks should do as much as possible to ensure they are ready to receive traffic before listening on the HTTP port.

## URL Space

> Note: URL definitions are found in [RFC 3986](https://tools.ietf.org/html/rfc3986#section-3).

The framework's web server must listen to ingress HTTP requests defined with the following URL scheme using the `PORT` specified above:

`https://hostname:$PORT`

For example, the hostname of a process running the functions framework running the function `my-function` at `localhost` on `8080` would result in the URL:

`https://localhost:8080`

Requesting this URL would invoke the function `$FUNCTION_TARGET`.

In a different example, a service using the Functions Framework could have the url `us-central1-my-gcp-project.cloudfunctions.net/my-function`.

> Note that the framework must listen to all inbound paths (`.*`) and route these requests to the function, with the following exceptions:
> 
> - `/robots.txt` - must respond with 404 without invoking function
> - `/favicon.ico` - must response with 404 without invoking function

## Supported Function Types

The framework **must** support at least the `http` and `cloudevents` function types. The framework **may** support legacy-style `event` function types. In statically typed languages, the framework must include an additional function type whereby the user can define unmarshalling rules. These function types dictate:

- The steps taken by the framework in response to ingress requests
- The function signatures which developers must adhere to when writing functions for use with a Functions Framework
- The mechanism by which the developer signals that the function has completed performing useful work

The developer must signal the function's signature type to the framework. This enables the framework to take appropriate unmarshalling, invocation and completion signalling steps.

### HTTP Functions (Signature Type: `http`)

The Functions Framework **must** support signature type `http`.

When the container receives an ingress request, the framework must invoke the developer's function by passing a language-idiomatic representation of HTTP objects as arguments to the function. These objects must enable the developer to perform common HTTP tasks, such as inspecting the request's content encoding or headers. These objects should be accurate representations of the HTTP request received by the execution environment (i.e., path, body and headers should not be modified before passing them to the user's function).

### CloudEvents Functions (Signature Type: `cloudevents`)

The Functions Framework **must** support signature type `cloudevents`.

When the container receives an ingress request, the framework must invoke the developer's function by passing an object corresponding to a [CloudEvents type](https://github.com/cloudevents/spec/blob/master/spec.md). This object does not expose HTTP semantics to the developer's function. The framework must handle unmarshalling HTTP requests into the CloudEvents object that is passed to the developer's function, and should support both [binary and structured content modes](https://github.com/cloudevents/spec/blob/master/http-protocol-binding.md#3-http-message-mapping) for incoming HTTP CloudEvent requests.

The developer's function must either explicitly or implicitly signal that it has completed performing useful work. The function may explicitly signal this condition by explicitly returning. The function may implicitly signal this condition by simply evaluating until it reaches the end of the function's code block.

### Legacy Events (Signature Type: `event`)

The Functions Framework **may** support signature type `events`. This signature supports non-CloudEvent style events. Your function have must use the following signature:

- 1st parameter `data`
- 2nd parameter `context`

Where the parameters have the contents detailed in [background functions](https://cloud.google.com/functions/docs/writing/background#function_parameters).

## Configuration

The framework must allow the developer to specify (1) the port ($PORT) on which the web server listens, (2) the target function to invoke in response to ingress requests and (3) the function's signature type.

This configuration may be provided implicitly or explicitly. In some languages, the function's signature type can be inferred by via reflection, inspecting the developer's function signatures, annotations, and exports - using language-idiomatic methods that feel natural to developers. In some languages, the signature type may need to be explicitly signalled by the developer.

The framework may, for developer convenience, provide multiple mechanisms (for example, environment variables, command-line arguments, configuration files) for developers to specify configuration. If multiple methods are provided, the order of precedence must be clearly specified.

## Observability

The framework may provide built-in observability support. For example, the framework might automatically add Trace spans, collect profiling information or provide a logger.

If such integrations are provided, they must be clearly documented and users must be able to enable or disable them. Disabling these integrations should result in a graceful default experience.

## Function-to-app Converter

A function-to-app converter must be provided. This converter takes function code as input and creates app code as output. This app code must be buildable into a container using an app-to-container converter (such as a [Cloud Native Buildpack](https://buildpacks.io)).

A developer may opt to perform these steps manually rather than using the function-to-app converter. For this reason, it is important that the steps taken by the function-to-app converter are idiomatic to the programming language selected for the Functions Framework. For example, a function-to-app converter written for a Python Functions Framework should expose a WSGI app which can be used by multiple HTTP servers such as gunicorn. A Go function-to-app converter would add a dependency via a `go.mod` file and add a boilerplate `main.go` file.

Example steps taken by the function-to-app converter may include:
- directory layout
- injecting a dependency into a dependencies file
- creating a boilerplate main package

## Stdout/Stderr and Logging expectations

Application logs to stdout/stderr within application code and logs from the Functions Framework itself are expected to appear in stdout/stderr of the process running the Functions Framework.

## HTTP Status Codes

The framework should return 4XX HTTP status error codes for client errors and 5XX HTTP status error codes for Function Framework errors according to [RFC 7231](https://tools.ietf.org/html/rfc7231#page-47). Examples:

- [4xx](https://tools.ietf.org/html/rfc7231#section-6.5): External errors from clients of the Functions Framework
  - [400](https://tools.ietf.org/html/rfc7231#section-6.5.1): Bad Request (i.e. malformed CloudEvent)
  - [429](https://tools.ietf.org/html/rfc6585#section-4): Too many requests to the framework
- [5xx](https://tools.ietf.org/html/rfc7231#section-6.6): Internal errors from the Functions Framework itself
  - [504](https://tools.ietf.org/html/rfc7231#section-6.6.5): Function timeout
