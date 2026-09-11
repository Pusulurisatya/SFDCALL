# Connecting Salesforce and Jira to an AI Agent Session

This note documents how to connect a Salesforce org and Jira when working
in an AI coding agent session (e.g. Claude Code on the web), where the
session runs in a headless remote container with no local browser.

## Salesforce CLI

Install the CLI:

```bash
npm install -g @salesforce/cli
```

### Connecting an org

`sf org login web` requires an interactive browser and a localhost OAuth
callback, so it only works on a machine where you can actually open a
browser (your own laptop/desktop). It will fail in a headless container
with `CannotOpenBrowserError`.

To connect an org from a headless session, authenticate locally first,
then bring the resulting credential over:

1. On a machine with a browser:
   ```bash
   npm install -g @salesforce/cli
   sf org login web -a myorg
   sf org auth show-sfdx-auth-url -o myorg
   ```
2. Copy the printed `force://...` URL. Treat it as a credential (it embeds
   a refresh token) — do not commit it or leave it in shell history.
3. In the headless session:
   ```bash
   sf org login sfdx-url --sfdx-url-file <path-to-file-containing-the-url> -a myorg
   ```

Alternative for headless-only environments: JWT bearer flow, which needs a
Connected App configured in the org with a self-signed certificate:

```bash
sf org login jwt --username <user> --jwtkeyfile <key.pem> --clientid <consumer-key> --instance-url <url>
```

### Network policy note

Some remote execution environments restrict outbound egress to an
allowlist of domains. If `sf org login` fails with a connection error
(not an auth error), check whether `*.salesforce.com` / `*.force.com` /
`*.my.salesforce.com` are allowed by the environment's network policy —
this is a per-environment setting, not something fixable from inside a
session.

## Jira (via MCP)

No local CLI setup is needed. When the Atlassian MCP server (`Atlassian_Rovo`
or equivalent) is connected to the session, Jira issues can be read,
created, updated, commented on, and transitioned directly by the agent —
see `writing-code/generate-feature.md` (or `generate-feature.md`) for the
prompt chain that uses this to turn a Jira ticket into implemented code.
