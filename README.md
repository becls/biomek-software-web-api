# Biomek Software Web API

This repository contains public documentation and reference materials for the **Biomek Software Web API**, a REST API for remotely monitoring and
controlling the Biomek software from another computer or automation controller. It is available in Biomek Software 6.0.5 and newer, when the
Biomek Software Web API Service (BiomekWebApi.Service) is installed and running as a Windows service on the Biomek workstation. It exposes endpoints
that clients can use to discover instruments, query their status, and integrate Biomek workflows with other systems.

## Quick start

**Connecting**: The service is accessible by default at `http://{host}:8080/`, where `{host}` is the name or IP address of the PC running Biomek Software. The administrator should 
reconfigure this port away from the publicly known default port. The API does not currently support HTTPS, so it is also highly recommended that you isolate the Biomek Software 
Web API as much as possible and use it on a trusted local network only.  Do not expose this API to the public internet.

**Authentication**: If the service is configured to require authentication, include the `X-Api-Key` header on every request.
See the API reference for details.

**Typical usage sequence**:

1. `GET /instruments` &mdash; discover which instruments are available.
2. `GET /instruments/{instrumentName}` &mdash; confirm the instrument is in a `ready` state before attempting a run.
3. Open the method to run, using one of the following, depending on whether the method's JSON content or the
   method's name in the workspace is available to the integrator:
   - 3a. `POST /instruments/{instrumentName}/methods` &mdash; open a JSON method sent inline in the
     request body, for integrators supplying the method's content directly rather than relying on one already
     present in the workspace. This also opens the method in the editor, so step 3b below is not needed afterward.
   - 3b. `POST /instruments/{instrumentName}/methods/{methodName}/open` &mdash; open a method that is
     already available in the workspace currently open in the editor. Not needed if step 3a was already used to
     open the method.
4. (Optional) `GET /instruments/{instrumentName}/variables` &mdash; list the method's Start step variables and their default values to
   discover what can be parameterized before starting the run.
5. `POST /instruments/{instrumentName}/run` &mdash; start running the currently open method. Optionally supply a `variables` object in
   the request body to override Start step variable values for this run, parameterizing the run without editing the method. See the API
   reference for the request shape and validation rules.
6. (Optional) `POST /instruments/{instrumentName}/pause` &mdash; pause a run in progress.
   `POST /instruments/{instrumentName}/resume` resumes it. Both endpoints enforce a
   per-instrument cooldown to prevent rapid cycling.
7. (Optional) `POST /instruments/{instrumentName}/abort` &mdash; abort a run that is running or paused.
8. `GET /instruments/{instrumentName}/events` &mdash; subscribe to the SSE stream to monitor run progress, monitor open dialogs,
   and detect completion.
9. (Optional) `GET /instruments/{instrumentName}/dialogs/{dialogId}/image` &mdash; Get the image displayed on an
   open dialog in order to correctly respond to the dialog.
10. (Optional) `POST /instruments/{instrumentName}/dialogs/{dialogId}/respond` &mdash; Respond to an open dialog
   in order to dismiss it and continue a run.

## Viewing the docs

The rendered API reference is published via GitHub Pages at: https://becls.github.io/biomek-software-web-api.

You can also browse the docs locally from a clone of this repository &mdash; see [Contents](#contents) below.

## Contents

- [`docs/`](docs/) &mdash; Contains reference documentation for the Biomek Software Web API and the Biomek Software JSON format.
  - [`openapi.json`](docs/openapi.json) is the OpenAPI 3.1 specification of the API's current endpoints, request/response shapes,
    and error model. It is generated directly from the service implementation, so it always matches the running service.
  - [`index.html`](docs/index.html) is a Scalar API reference page that renders `openapi.json` in the browser. It is fully
    self-contained &mdash; the Scalar renderer is bundled with the docs in the `vendor/` folder, so the page loads no external CDNs or fonts
    and works offline. It must be served over HTTP &mdash; opening it directly via `file://` will not work because the browser cannot fetch the spec under that
    scheme. From a clone of this repo:

    ```sh
    cd docs
    python -m http.server 8000
    ```

    then open <http://localhost:8000/>. Any other static file server works equivalently.
  - [`AdminToolGuide.md`](docs/AdminToolGuide.md) explains what the **Biomek Software Web API Admin Tool** is, what it controls,
    and how to use it safely as a site administrator.
  - [`json-format/`](docs/json-format/) contains reference material for the **Biomek Software JSON format** in plain text documents that
    can provide context to an AI or an automation engineer.

## Feedback

Please open an issue on this repository for documentation bugs, missing examples, or suggestions for additional content.
