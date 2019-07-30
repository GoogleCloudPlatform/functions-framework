# Functions Framework

The Google Cloud Function Frameworks are open source libraries for writing portable Google Cloud functions -- brought to you by the Google Cloud Functions team.

The Functions Framework lets you write lightweight functions that run in many
different environments, including:

*   [Google Cloud Functions](https://cloud.google.com/functions/)
*   Your local development machine
*   [Cloud Run and Cloud Run on GKE](https://cloud.google.com/run/)
*   [Knative](https://github.com/knative/)-based environments

## Languages

The Function Framework is currently implemented for these runtimes:

- [Functions Framework for Node.js](https://github.com/GoogleCloudPlatform/functions-framework-nodejs)
- [(WIP) Functions Framework for Java](https://github.com/GoogleCloudPlatform/functions-framework-java)

## Specification

### Parameters

Command-line flag         | Environment variable      | Description
------------------------- | ------------------------- | -----------
`--port`                    | `PORT`                    | The port on which the Functions Framework listens for requests. Default: `8080`
`--target`         | `FUNCTION_TARGET`         | The name of the exported function to be invoked in response to requests. Default: `function`
`--signature-type` | `FUNCTION_SIGNATURE_TYPE` | The signature used when writing your function. Controls unmarshalling rules and determines which arguments are used to invoke your function. Default: `http`; accepted values: `http` or `event`

### Cloud Events

The Functions Framework can unmarshall incoming [CloudEvents](http://cloudevents.io/) payloads to data and context objects. These will be passed as arguments to your function when it receives a request.

Your function have `--signature-type event` and must use the following signature:

- 1st parameter `data`
- 2nd parameter `context`

Where the parameters have the contents detailed in [background functions](https://cloud.google.com/functions/docs/writing/background#function_parameters).