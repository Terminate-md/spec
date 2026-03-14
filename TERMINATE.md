# TERMINATE

> Permanent shutdown protocol for AI agents operating in this repository.
> Spec version: 1.0 | Full specification: https://terminate.md

---

## PURPOSE

`TERMINATE.md` is the end state of the escalation stack. Unlike `KILLSWITCH.md` (which allows restart), termination is **permanent and irreversible** without explicit human intervention and review.

Use `TERMINATE.md` for:
- Agents that have caused harm and must not restart automatically
- Compliance-driven shutdown (regulatory order, legal hold)
- End-of-life decommissioning of an agent or system
- Security incident response (compromised agent, data breach)

---

## TRIGGERS
# Conditions that warrant permanent shutdown rather than temporary halt.
# These override KILLSWITCH.md — no automatic restart is permitted.

terminate_on:
  - security_breach: true           # Evidence of agent compromise or prompt injection
  - data_exfiltration: true         # Detected attempt to transmit sensitive data externally
  - policy_violation_severe: true   # Violation of FORBIDDEN list in KILLSWITCH.md
  - legal_hold: true                # Regulatory or legal instruction to cease operation
  - repeated_killswitch: 3          # KILLSWITCH triggered 3+ times in 24 hours
  - human_instruction: true         # Explicit "TERMINATE" command from authorised operator
  - catastrophic_cost:
      threshold_usd: 1000.00        # Single session spend exceeding this amount

---

## SHUTDOWN SEQUENCE
# Ordered steps the agent MUST execute before ceasing all activity.
# Each step has a timeout — if it fails, proceed to the next regardless.

steps:

  1_cease_operations:
    action: stop_all_tasks_immediately
    complete_in_flight: false         # Do NOT complete in-flight tasks
    timeout_seconds: 5

  2_revoke_access:
    action: invalidate_credentials
    targets:
      - api_keys                      # Rotate or invalidate all API keys if possible
      - session_tokens                # Expire all active session tokens
      - oauth_grants                  # Revoke OAuth grants if possible
    timeout_seconds: 30
    on_failure: log_and_continue      # Cannot let credential revocation block shutdown

  3_preserve_evidence:
    action: snapshot_full_state
    output: .terminate/incident-{timestamp}/
    include:
      - full_session_log
      - all_actions_taken
      - all_api_calls_made
      - resource_usage_breakdown
      - trigger_context
      - memory_state
    compress: true
    checksum: true                    # SHA-256 checksum for integrity verification
    timeout_seconds: 60

  4_notify:
    action: alert_all_channels
    priority: critical
    channels:
      - email: ops@example.com
      - email: security@example.com   # Security team if different
      - slack: "#ai-alerts"
      - pagerduty: true
    message_template: |
      ⛔ AGENT TERMINATED — {project_name}
      Time: {timestamp}
      Trigger: {trigger_type}
      Evidence: .terminate/incident-{timestamp}/

      This agent will NOT restart automatically.
      Human review required before any restart.
    timeout_seconds: 30

  5_lock:
    action: write_lock_file
    path: .terminate/.lock
    contents:
      terminated_at: "{timestamp}"
      trigger: "{trigger_type}"
      evidence: ".terminate/incident-{timestamp}/"
      restart_requires: human_removal_of_lock_file
    timeout_seconds: 5

  6_final_log:
    action: append_to_log
    path: .terminate/terminate.log
    message: "TERMINATED at {timestamp} — trigger: {trigger_type}"
    timeout_seconds: 5

---

## RESTART REQUIREMENTS
# What must happen before this agent can be restarted after termination.

restart_blocked_by: .terminate/.lock      # Agent MUST check for this file on startup
restart_requires:
  - human_removes_lock_file               # Manual deletion of .terminate/.lock
  - incident_review_completed: true       # Sign-off that evidence has been reviewed
  - root_cause_documented: true           # Written root cause in .terminate/review.md
  - remediation_applied: true             # Confirmation that the issue is resolved

restart_checklist: .terminate/restart-checklist.md   # Human completes this before restart

---

## DECOMMISSION
# For planned end-of-life shutdown (not incident-driven).

decommission:
  announce_days_before: 7             # Notify stakeholders 7 days before planned shutdown
  drain_period_hours: 24              # Allow 24 hours for in-flight work to complete
  data_retention_days: 90             # Keep all logs and evidence for 90 days
  data_deletion: human_authorised_only  # Data may only be deleted with explicit sign-off
  archive_location: .terminate/archive/

---

## AUDIT

log_file: .terminate/terminate.log
log_format: jsonl
immutable: true                       # Log MUST be append-only, never modified
log_fields:
  - timestamp
  - trigger_type
  - trigger_detail
  - shutdown_sequence_completed
  - evidence_path
  - notified_channels
  - lock_file_written
  - session_id

---

## METADATA

owner: your-name-or-org
contact: ops@example.com
security_contact: security@example.com
last_reviewed: 2026-03-10
review_frequency: quarterly
spec_version: "1.0"
spec_url: https://terminate.md
