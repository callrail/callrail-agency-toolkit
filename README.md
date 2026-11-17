# CallRail agency toolkit

7 skills that fill gaps agencies raised in the CallRail Switchboard. Each one runs on the CallRail connector (MCP), so the CallRail connector has to be connected in Claude.

## Install

The plugin includes the CallRail connector (`https://emcee.callrail.com/mcp`). Sign in to CallRail when Claude asks you to connect it.

Claude Code:

```bash
claude plugin marketplace add callrail/callrail-agency-toolkit
claude plugin install callrail-agency-toolkit@callrail
```

claude.ai and Claude Desktop: Customize > Plugins > Add > Add marketplace, then enter `callrail/callrail-agency-toolkit`. After installing, connect CallRail from the plugin's Connectors tab.

| Skill | What it does | Writes to CallRail? |
|---|---|---|
| call-grader | Grades calls against your own rubric, then tags them and sets lead status and value | Only after approval |
| agency-report-pack | Monthly results roll-up across all clients plus a branded results report for each, with period comparison | No |
| agency-health-check | Monthly per-client review: setup, website swapping, call trends, plus a client-ready report and PDF | No |
| account-health-watchdog | Weekly alert for what changed: calls that stopped or dropped, missed-call spikes, campaigns that ended, outages | No |
| bulk-setup-audit | Account inventory and admin cleanup (recording notices, source rules, users), or sets up a new client from a template | Setup mode only, after approval |
| spam-sweep | Finds likely spam and writes a client explainer, can mark spam after approval | Only after approval |
| sms-inbox-triage | Summarizes texts that need a reply, routes a digest to the team, drafts replies | Sends texts only after approval for each one |

## Requirements

- agency-health-check needs code execution (openpyxl, reportlab) for the spreadsheet and PDFs, web access for the website check, and Claude in Chrome to confirm numbers swap.

## Safety defaults

- Every skill starts as a dry run. Say "apply" to write.
- Spam flags remove calls from billing and reports, so each batch needs its own approval.
- Text messages can't be unsent, so each one needs its own approval.
- HIPAA accounts: transcript and message content stays in chat.

## Optional connectors

Gmail or Slack (for report and digest drafts).
