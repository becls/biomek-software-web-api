# Biomek Software Web API Admin Tool Guide

This guide explains what the **Biomek Software Web API Admin Tool** is, what it controls, and how to use it safely as a site administrator.

The Admin Tool is the recommended way to manage the installed Biomek Software Web API service configuration on a workstation. It lets you:

- Set the HTTP port used by the service.
- Check whether the service is running and reachable.
- Configure the Windows Defender Firewall rule for inbound API traffic.
- Create, rename, and deactivate API keys.
- Turn API key requirement on or off.
- Open the service and admin audit log folder for troubleshooting.

## Admin Tool Layout

The app has two sections in the left navigation:

- **Settings**
  - Port Configuration
  - Windows Defender Firewall Control
  - Result panel (action outcomes, endpoint status, timestamp, and details)
- **API Keys**
  - Key list and status
  - Create key workflow
  - Require API key toggle

## Common Tasks

### 1) Change the API HTTP Port

1. Open **Settings**.
2. In **Port Configuration**, enter a port from `1` to `65535`.
3. Select **Save Port**.

What happens:

- The new port is saved to the Biomek Software Web API service configuration file.
- The tool checks service state and endpoint reachability.
- The Windows Defender Firewall rule port is synchronized to match the saved HTTP port.

Notes:

- If the port value is invalid, the tool will ask for a valid port range.
- If configuration write fails, the Result panel shows the error.

### 2) Check Service Reachability

1. Open **Settings**.
2. Select **Check Endpoint**.

What this check means:

- The tool probes `http://127.0.0.1:{port}/` with a short timeout.
- Any HTTP response counts as reachable, including `401 Unauthorized`.
  - This is expected when authentication is enabled but no key is supplied.

The check is for connectivity/listening status, not authorization success.

### 3) Allow or Disallow Inbound Firewall Access

1. Open **Settings**.
2. In **Windows Defender Firewall Control**, choose:

- **Allow**: create and/or enable the inbound rule.
- **Disallow**: disable the rule without deleting it.
- **Refresh**: reload current firewall status.

Key behavior:

- Rule name: `BiomekWebApi.HTTP.Inbound`
- Protocol: TCP inbound
- Profiles: Domain and Private by default
- Public profile is disabled

If your organization uses another firewall product, configure that product separately to allow the configured port for Biomek Software Web API.

### 4) Create a New API Key

1. Open **API Keys**.
2. Select **Create new API key**.
3. In the dialog, copy the generated key and store it securely.
4. Enter a unique **Key name**.
5. Select **Activate key**.

Important:

- The plaintext key is shown once during creation and cannot be displayed again later.
- Do not share keys or embed them in browser/client-side code.

### 5) Deactivate an API Key

1. Open **API Keys**.
2. Select the key row.
3. Select the deactivate (trash) action.
4. Confirm the warning.

Effects:

- Deactivation is permanent (cannot be undone).
- Clients using that key are denied access.
- Open event streams for that key are closed.

### 6) Turn API Key Requirement On or Off

1. Open **API Keys**.
2. Use the **Require API key** checkbox.

If you turn it **off**, the tool shows a warning confirmation because this allows unauthenticated network access to API operations.

Recommendation:

- Keep **Require API key** enabled in production unless your environment has compensating controls and explicit approval.

### 7) Open Log Folder

1. Open **Settings**.
2. In the Result panel, select **Open Log Folder**.

Location:

- `%ProgramData%\Beckman Coulter\BiomekWebApi\Logs\`

This folder contains:

- Biomek Software Web API Service logs
- Admin Tool audit logs (`AdminTool-<date>.log`)

Important:

- Log files are restricted to Administrators.
- To open logs in a text editor, launch that editor as Administrator first (for example, run Notepad as Administrator, then open the log file).

## 8) Security and Operational Notes

- The Admin Tool requires elevation to ensure only authorized admins can change network/security settings.
- Administrative actions are audit logged (actor, machine, action, success/failure).
- API key secrets are stored encrypted in configuration; plaintext is not persisted by the UI.

## Quick Setup Checklist for New Integrations

1. Confirm service is installed and running.
2. Set/verify HTTP port.
3. Enable firewall access for the selected port.
4. Make sure the service machine has private or domain network profile.
5. Create an API key and distribute it securely to the integration client.
6. Keep `Require API key` enabled.
7. Validate connectivity with a simple request to the service root.
