# Collector API documentation

Public documentation for agents and integrations using the Collector External HTTP API. This repository is intended to be usable without access to the application's source code.

## Status

Repository initialized. The generated API specification, endpoint reference and application-side synchronization/CI checks are not yet installed. This is not yet a complete API contract.

Start with [AGENTS.md](AGENTS.md). Obtain the deployment base URL and an authorized API key separately from the operator.

## Publication boundary

This repository must not contain application source code, credentials, internal configuration, real survey content, respondent information or responses. Examples must use synthetic data and credential placeholders.

The planned integration generates the API reference from the application and checks it for drift locally and in CI. The application will pin this repository as a Git submodule. Generated files will not be edited manually; publication must be reviewed before pushing.
