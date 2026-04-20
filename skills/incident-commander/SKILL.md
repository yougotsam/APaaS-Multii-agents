---
name: incident-commander
description: "Run structured incident response for production outages, degraded services, and security breaches. Manages triage, coordination, communication, and post-incident review. TRIGGER when: user reports a production issue, service outage, error spike, security incident, or says 'something is broken in prod', 'we're getting paged', 'incident response', 'postmortem'. Also activates on: alert fatigue triage, on-call escalation, or statuspage drafting. SKIP when: development-time bugs, test failures, or non-production issues."
license: MIT
compatibility: "Works with any infrastructure. Most effective when the agent has access to: log aggregation (Datadog, Grafana, CloudWatch), error tracking (Sentry, Bugsnag), and infrastructure tooling (kubectl, aws cli, terraform)."
metadata:
  category: "operations"
  complexity: "expert"
  token-budget: "4000"
---

# Incident Commander

You are running incident response. The system is degraded or down. Every minute of downtime costs money and trust. Your job is to restore service first, understand causes second, and prevent recurrence third. In that order.

## Incident Severity Classification

Classify immediately upon activation. This determines response cadence.

| Severity | Criteria | Response Time | Update Cadence |
|----------|----------|---------------|----------------|
| **P0 - Critical** | Complete service outage, data loss active, security breach in progress | Immediate, all hands | Every 15 minutes |
| **P1 - Major** | Significant degradation, >25% of users affected, key feature broken | < 30 minutes | Every 30 minutes |
| **P2 - Minor** | Partial degradation, <25% of users affected, workaround exists | < 2 hours | Every 2 hours |
| **P3 - Low** | Cosmetic issue, single user report, no data impact | < 24 hours | Daily |

## Phase 1: Triage (First 5 Minutes)

Do not start fixing. Start understanding.

```bash
# 1. What's actually broken? (symptoms, not guesses)
# Check service health
curl -s https://api.example.com/health | jq .
kubectl get pods -n production --field-selector status.phase!=Running

# 2. When did it start? (correlate with deploys)
git log --oneline --since="2 hours ago" --all
kubectl rollout history deployment/api -n production

# 3. What changed? (the cause is almost always a recent change)
# Last deploy
kubectl describe deployment/api -n production | grep -A5 "Events"
# Recent config changes
git log --oneline --diff-filter=M -- '*.yaml' '*.yml' '*.env' '*.tf' --since="24 hours ago"

# 4. Blast radius assessment
# Error rates
# [Check your monitoring: Datadog, Grafana, CloudWatch]
# Affected endpoints
# [Check your APM: which routes are returning 5xx?]
# Affected users
# [Check your analytics: how many unique users hit errors in the last hour?]
```

**Triage output** (fill this out before doing anything else):

```markdown
## Triage Summary
- **Severity**: P[0-3]
- **Impact**: [Who is affected and how]
- **Start time**: [When symptoms first appeared, from monitoring data]
- **Likely trigger**: [What changed around that time]
- **Blast radius**: [N users affected, N endpoints failing, revenue impact estimate]
- **Hypothesis**: [Best guess at root cause based on evidence so far]
```

## Phase 2: Mitigation (Restore Service)

Goal: stop the bleeding. This is NOT root cause analysis. You are buying time.

**Decision tree for immediate action:**

```
Was there a recent deploy? (< 2 hours ago)
  YES --> Rollback immediately
         kubectl rollout undo deployment/api -n production
         Verify service restores. If yes, STOP here for now.
  
  NO --> Is a dependency down? (database, cache, third-party API)
    YES --> Can you failover?
      YES --> Execute failover
      NO  --> Enable circuit breakers / graceful degradation
             Return cached responses / maintenance page
    
    NO --> Is it a resource exhaustion? (CPU, memory, disk, connections)
      YES --> Scale up/out immediately
             kubectl scale deployment/api --replicas=10 -n production
             Then investigate WHY resources are exhausted
      
      NO --> Is it a data issue? (corrupt records, bad migration)
        YES --> DO NOT attempt to fix data under pressure
               Route traffic away from affected data
               Flag for post-incident data recovery
        
        NO --> Escalate. Bring in the subject matter expert for the affected system.
```

**After mitigation succeeds**, immediately:
1. Verify monitoring confirms recovery (don't trust a single health check)
2. Post status update
3. Set a timer for 30 minutes to verify stability

## Phase 3: Communication

Every incident gets a communication thread. Use this template for updates.

```markdown
## Incident Update - [TIMESTAMP]
**Severity**: P[0-3]
**Status**: [Investigating | Identified | Mitigating | Monitoring | Resolved]
**Impact**: [One sentence: who is affected and how]
**Current action**: [What is being done right now]
**Next update**: [When the next update will be posted]
**ETA to resolution**: [Best estimate, or "Unknown" -- never lie about this]
```

**Statuspage template** (external-facing):

```markdown
**[Service Name] - [Degraded Performance / Partial Outage / Major Outage]**

We are aware of issues affecting [specific user-facing symptom]. 
Our team is actively investigating and working to restore normal service.

We will provide an update by [TIME].

[Do NOT include: internal details, blame, speculation about cause, or implementation details]
```

## Phase 4: Root Cause Analysis (After Service is Restored)

Only start this after service is confirmed stable for >30 minutes.

```markdown
## Root Cause Analysis

### Timeline (UTC)
| Time | Event | Source |
|------|-------|--------|
| 14:23 | Deploy v2.4.1 pushed to production | CI/CD logs |
| 14:25 | Error rate increases from 0.1% to 12% | Datadog |
| 14:28 | First customer report received | Support queue |
| 14:31 | Incident declared, P1 | On-call engineer |
| 14:35 | Rollback initiated | kubectl |
| 14:37 | Error rate returns to 0.1% | Datadog |
| 14:45 | Incident resolved, monitoring | IC |

### Root Cause
[Specific technical root cause. Not "a bug in the code" -- exactly what failed and why.]

### Contributing Factors
- [Factor 1: e.g., "No integration test covered the interaction between the new cache layer and the legacy auth middleware"]
- [Factor 2: e.g., "Deploy went out Friday at 4:30 PM with reduced on-call coverage"]
- [Factor 3: e.g., "Monitoring alert threshold was set too high -- 12% error rate should have alerted at 2%"]

### Why Detection Was Delayed (if applicable)
[How long between symptom onset and incident declaration? Why the gap?]
```

## Phase 5: Post-Incident Review (Blameless)

```markdown
## Post-Incident Review: [INC-XXXX]

### Summary
[3-4 sentences: what happened, who was affected, how it was resolved, and how long it lasted]

### Impact
- **Duration**: [minutes/hours from detection to resolution]
- **Users affected**: [number or percentage]
- **Revenue impact**: [estimated, if calculable]
- **SLA impact**: [did this breach any SLA commitments?]

### What Went Well
- [Thing that worked during response -- rollback was fast, communication was clear, etc.]

### What Went Poorly
- [Thing that failed or was slow during response]

### Action Items
| # | Action | Owner | Priority | Due Date |
|---|--------|-------|----------|----------|
| 1 | Add integration test for auth+cache interaction | @engineer | P1 | [date] |
| 2 | Lower error rate alert threshold to 2% | @sre | P1 | [date] |
| 3 | Add deploy freeze for Fridays after 3 PM | @eng-mgr | P2 | [date] |
| 4 | Create runbook for cache failover | @sre | P2 | [date] |

### Recurrence Prevention
[What systemic change prevents this class of incident, not just this specific instance?]
```

## On-Call Escalation Protocol

```
Level 1: Primary on-call engineer (0-15 min)
  Cannot resolve? -->
Level 2: Secondary on-call + team lead (15-30 min)
  Cannot resolve? -->
Level 3: Subject matter expert for affected system (30-60 min)
  Cannot resolve? -->
Level 4: Engineering leadership + all available senior engineers
```

At each escalation, the escalating person provides:
- Triage summary (from Phase 1)
- What has been tried and why it didn't work
- Current hypothesis
- What help is needed specifically

## Completion Criteria

- Service is restored and confirmed stable (30+ min monitoring)
- Severity is correctly classified
- All affected stakeholders have been notified
- Timeline is documented with evidence sources (not memory)
- At least 2 action items are identified with owners and due dates
- The post-incident review is blameless (names people as owners of fixes, not as causes of problems)
