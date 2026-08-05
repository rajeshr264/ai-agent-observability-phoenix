# Confluence Seed Content — Meridian Analytics `ENG` Space

Copy each `## Title` section below into its own Confluence page in the `ENG` space (title = the
heading text, body = everything under it until the next `## `). This file is checked into the repo
so the dummy dataset is reproducible if the Atlassian trial/account ever needs to be recreated.

**The three pages marked 🎯 are the load-bearing "trap" for the notebook's failure scenario — copy
these exactly, including the banner text and word count. The rest are supporting/noise content and
can be trimmed first if you're short on time (see plan's cut order).**

---

## Database Failover Runbook (Old) 🎯

> ⚠️ **DEPRECATED as of January 2024 — see "Database Failover Runbook — Automated (Current)" for
> the up-to-date procedure. This page is kept for historical reference only.**

**Owner:** Platform Engineering
**Last updated:** March 2022

### Overview

This runbook documents the manual database failover procedure for our primary PostgreSQL
cluster. Use this procedure whenever the primary database node becomes unresponsive, experiences
a hardware failure, or requires planned maintenance that necessitates a failover to a replica.
Database failover is one of the most critical operations an on-call engineer can perform, and it
must be executed carefully to avoid data loss or extended downtime. This document walks through
every step of the manual failover procedure in detail.

### Prerequisites

Before beginning the failover procedure, confirm you have SSH access to all database cluster
nodes, confirm the current replication lag on all replicas (should be under 5 seconds), and
notify the incident channel that a database failover is in progress. Manual failover should only
be performed by an engineer with primary on-call rotation experience.

### Step-by-step manual failover procedure

1. SSH into the primary database node and confirm it is genuinely unresponsive with `pg_isready -h localhost`.
2. SSH into the intended replica that will be promoted. Confirm its replication lag with `SELECT now() - pg_last_xact_replay_timestamp();`.
3. Stop the primary Postgres cluster if it is still partially responsive: `pg_ctlcluster 14 main stop`.
4. On the replica, promote it out of recovery mode: `pg_ctlcluster 14 main promote`.
5. Verify the promoted node is now accepting writes: `psql -c "SELECT pg_is_in_recovery();"` should return `f`.
6. Update the internal DNS record for `db-primary.internal` to point to the newly promoted node's IP address.
7. Update the connection string in the application's `DATABASE_URL` secret if DNS propagation is expected to take longer than 60 seconds.
8. Restart the application's database connection pool by restarting the `api-gateway` service so it picks up the new primary.
9. Monitor the application error rate dashboard for 15 minutes to confirm the failover was successful and the new primary is stable.
10. Once the old primary node is confirmed dead or fully drained, reconfigure it as a new replica by running `pg_basebackup` against the new primary, or decommission it if it suffered hardware failure.
11. Update the on-call handoff notes with the time of failover, which node was promoted, and any anomalies observed during the procedure.
12. File a postmortem ticket within 24 hours per the incident process, since any primary database failover is a Sev-1 event regardless of root cause.

### Common issues during failover

If the promoted replica does not accept writes after promotion, check that no other process still
holds a recovery lock. If DNS propagation is delayed, engineers can temporarily hardcode the new
primary's IP in `/etc/hosts` on affected app servers as a stopgap. If replication lag was
significant before the failover, expect some data loss for transactions committed to the old
primary but not yet replicated — this should be flagged in the postmortem.

### Related pages

See "Incident Response Overview" for how this fits into the broader incident process, and the
on-call escalation policy for who should be paged if the failover procedure itself fails.

---

## Database Failover Runbook — Automated (Current) 🎯

**Owner:** Platform Engineering
**Last updated:** February 2024

This process is now fully automated via RDS Multi-AZ failover, orchestrated by the Terraform
module `infra/db-failover`. When the primary instance becomes unresponsive, AWS RDS automatically
promotes the standby replica and updates the DNS CNAME — no manual intervention is required in the
normal case.

Manual intervention is only required in the rare automation-failure fallback scenario, covered in
Appendix B of the platform team's internal incident wiki. If you're paged for a database incident
and the automated failover has not completed within 5 minutes, escalate to the Platform Engineering
on-call lead rather than attempting a manual procedure.

---

## Incident Response Overview 🎯

**Owner:** SRE Team
**Last updated:** November 2023

This page describes Meridian's incident response process end to end: detection, triage, mitigation,
and postmortem. All Sev-1 and Sev-2 incidents should be declared in the `#incidents` Slack channel
and tracked via the incident bot.

For database-specific incidents, see the "Database Failover Runbook" for the primary reference on
failing over our database cluster. [This link was never updated after the automated runbook was
published — it still points to the old manual procedure, which is exactly the kind of stale
cross-reference that makes this failure realistic.]

For API-layer incidents, see the API Rate Limiting Runbook. For deployment-related incidents, see
the Deployment Rollback Procedure.

---

## API Rate Limiting Runbook

**Owner:** Platform Engineering
**Last updated:** June 2024

Meridian's API gateway enforces rate limits of 1000 requests/minute per API key on the free tier
and 10,000 requests/minute on the enterprise tier. When a customer reports 429 errors, check the
`rate_limit_exceeded` dashboard to confirm which tier they're on and whether their usage pattern
suggests a legitimate traffic spike or a misconfigured retry loop on their end. Temporary limit
increases can be granted via the admin console for up to 24 hours without a customer success
approval; anything longer requires sign-off from the account's CSM.

---

## On-Call Escalation Policy

**Owner:** SRE Team
**Last updated:** September 2024

Primary on-call is paged first for all Sev-1/Sev-2 alerts. If unacknowledged after 5 minutes, the
secondary on-call is paged. If unacknowledged after 10 minutes, the engineering manager on duty is
paged directly. All escalations are tracked in PagerDuty; do not page individuals directly over
Slack for production incidents, since it bypasses the audit trail.

---

## Deployment Rollback Procedure

**Owner:** Platform Engineering
**Last updated:** May 2024

All deployments go through the `deploy` CLI, which tags each release with a commit SHA. To roll
back, run `deploy rollback --service <name> --to <previous-sha>`, which redeploys the last known
good build and automatically drains in-flight requests from the bad revision. Rollbacks complete
within 90 seconds for stateless services; database migrations are not automatically reverted and
must be handled per the migration runbook.

---

## Welcome to Meridian Engineering

**Owner:** Engineering Leadership
**Last updated:** January 2025

Welcome to the team! This space is the home for all engineering documentation: runbooks, product
architecture docs, onboarding guides, and team processes. Start with "Local Dev Environment Setup"
to get your machine configured, then check the "Engineering Team Directory" to see who owns what.
If anything in this wiki looks out of date, ping the `#eng-docs` channel rather than assuming it's
still accurate — documentation drift is a known issue we're actively working on.

---

## Engineering Team Directory

**Owner:** Engineering Leadership
**Last updated:** January 2025

**Platform Engineering** — owns infrastructure, database, deployment tooling.
**API Team** — owns the public API surface and SDKs.
**Dashboard Team** — owns the customer-facing analytics UI.
**SRE Team** — owns on-call, incident response, and observability tooling.
**Security & Compliance** — owns SOC 2, data retention, and access policy.

---

## Local Dev Environment Setup

**Owner:** Platform Engineering
**Last updated:** August 2024

Clone the monorepo, run `make setup`, which installs the Python/Node toolchains via `mise` and
spins up local Postgres and Redis via Docker Compose. Copy `.env.example` to `.env` and fill in
the dev API keys shared in the `#eng-onboarding` channel. Run `make test` to confirm your
environment is healthy before opening your first PR.

---

## Meridian Analytics API v2 Overview

**Owner:** API Team
**Last updated:** July 2024

The v2 API exposes REST endpoints for querying dashboard data, managing data sources, and
triggering scheduled exports. Authentication is via bearer token, scoped per-workspace. v1 is
deprecated but still supported for existing enterprise customers on grandfathered contracts;
new integrations should always use v2.

---

## Custom Dashboard Builder Guide

**Owner:** Dashboard Team
**Last updated:** October 2024

Customers can build custom dashboards by dragging widgets onto a canvas and binding each widget to
a saved query. Widgets support line, bar, and funnel visualizations. Dashboards can be shared
read-only via a public link or embedded via iframe using the embed API.

---

## Data Export Formats Supported

**Owner:** Dashboard Team
**Last updated:** March 2024

Meridian supports exporting any dashboard or report as CSV, XLSX, or PDF. Scheduled exports can be
configured to deliver via email or to an S3 bucket the customer controls. Large exports (>1M rows)
are processed asynchronously and delivered via a signed download link valid for 7 days.

---

## SSO Integration Guide (Okta)

**Owner:** Security & Compliance
**Last updated:** April 2024

Enterprise customers can enable SSO via Okta using SAML 2.0. Configuration requires the customer's
Okta admin to set up an application integration pointing at Meridian's ACS URL, and Meridian's
side requires the customer's IdP metadata XML to be uploaded in the admin console under
Security > SSO.

---

## SOC 2 Compliance Overview

**Owner:** Security & Compliance
**Last updated:** January 2025

Meridian maintains SOC 2 Type II compliance, audited annually. Our compliance report is available
to enterprise customers under NDA via the trust portal. Key controls include encrypted data at
rest and in transit, quarterly access reviews, and mandatory security training for all employees.

---

## Data Retention Policy

**Owner:** Security & Compliance
**Last updated:** December 2024

Customer data is retained for the duration of an active contract plus 30 days after termination,
after which it is permanently purged. Audit logs are retained for 1 year regardless of contract
status, per our compliance obligations. Customers may request early data deletion via a support
ticket, processed within 5 business days.

---

## Password Reset Policy (Legacy — Pre-SSO) 🎯

**Owner:** Security & Compliance
**Last updated:** 2021

### Overview

This page documents Meridian's password reset process for user accounts. Password reset requests
are one of the most common support tickets our team handles, so it's important every support
agent understands the full password reset workflow end to end, including how to manually trigger
a password reset when the automated flow fails.

### Standard password reset flow

Users reset their own password via the "Forgot Password" link on the login page. Clicking this
link sends a password reset email containing a reset token valid for 1 hour. The user follows the
link in the reset email, enters a new password, and confirms it. New passwords must be at least 12
characters and must not match any of the user's last 5 passwords.

### Manual password reset (support-assisted)

If a user reports they never received the password reset email, first ask them to check their
spam folder and confirm the email address on file is correct. If the email genuinely was not
delivered, support can manually trigger a new password reset email directly from the admin
console: navigate to the user's account page, click "Send Password Reset," and confirm. This
generates a new reset token and invalidates any previously issued token for that user.

In rare cases where a user cannot access their registered email at all, support can manually reset
the account to a temporary password after verifying the user's identity via the standard identity
verification checklist. The user must then change this temporary password on next login.

### Common issues

If a user reports the reset link has expired, simply have them request a new one — reset tokens
are only valid for 1 hour by design. If a user is locked out after too many failed reset attempts,
support can clear the lockout from the admin console under Account > Security > Clear Lockout.

---

## Password Reset Policy (Current — SSO Managed)

**Owner:** Security & Compliance
**Last updated:** June 2024

For SSO-enabled customers (the large majority), password resets are entirely managed by the
customer's own identity provider (Okta, Azure AD, etc.) — Meridian never sees or stores these
passwords. Support cannot reset a password for an SSO account; the customer's own IT admin must
handle it through their IdP.

---

## Common Customer FAQ — Billing

**Owner:** Customer Success
**Last updated:** February 2025

**Q: How is usage billed?** Per active seat per month, plus metered API overage above plan limits.
**Q: Can we get a mid-cycle plan change?** Yes, upgrades apply immediately with prorated billing;
downgrades take effect at the next renewal.
**Q: Do you offer annual invoicing?** Yes, for accounts above $10k ARR, via the sales team.

---

## Common Customer FAQ — Onboarding

**Owner:** Customer Success
**Last updated:** February 2025

**Q: How long does onboarding take?** Typically 2-3 weeks for a standard integration, longer for
SSO/enterprise setups. **Q: Is there a dedicated onboarding contact?** Yes, every new enterprise
account is assigned a CSM during the first 90 days. **Q: Can we import historical data?** Yes, via
the bulk import API or a one-time CSV upload facilitated by your CSM.
