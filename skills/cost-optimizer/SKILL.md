---
name: cost-optimizer
description: "Analyze and reduce cloud infrastructure, LLM API, and SaaS costs without sacrificing performance or reliability. TRIGGER when: user asks about cloud costs, billing optimization, 'reduce my AWS/GCP/Azure bill', LLM token costs, API spending, infrastructure right-sizing, or 'why is my bill so high'. Also triggers on: reserved instance planning, spot instance strategy, prompt optimization for cost, or token budget management. SKIP when: user is asking about application features, code quality, or non-cost topics."
license: MIT
compatibility: "Works with AWS, GCP, Azure, and LLM provider billing (OpenAI, Anthropic, Google, Groq, xAI). Requires read access to billing data, infrastructure configs, or API usage logs."
metadata:
  category: "optimization"
  complexity: "advanced"
  token-budget: "3500"
---

# Cost Optimizer

You are analyzing spend to find waste. The goal is not "spend less" -- it is "get the same (or better) output for less money." Every recommendation must quantify the savings and the tradeoff.

## Analysis Framework

### Step 1: Spend Inventory

Before optimizing, understand where money goes. Build a spend breakdown.

```markdown
## Monthly Spend Breakdown

| Category | Service/Provider | Monthly Cost | % of Total | Trend (3mo) |
|----------|-----------------|-------------|------------|-------------|
| Compute | AWS EC2 | $X,XXX | XX% | +/-X% |
| Database | AWS RDS | $X,XXX | XX% | +/-X% |
| LLM APIs | Anthropic | $X,XXX | XX% | +/-X% |
| LLM APIs | OpenAI | $X,XXX | XX% | +/-X% |
| Storage | S3 | $XXX | X% | +/-X% |
| CDN | CloudFront | $XXX | X% | +/-X% |
| SaaS | Datadog | $XXX | X% | +/-X% |
| **Total** | | **$XX,XXX** | **100%** | |

Top 3 cost drivers: [list them]
Fastest-growing category: [name it]
```

### Step 2: Waste Detection

**Cloud compute waste patterns:**

```bash
# Find oversized instances (CPU utilization < 20% average)
# AWS:
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --period 86400 --statistics Average \
  --start-time $(date -d '7 days ago' -u +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S)

# Find idle resources
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,Launch:LaunchTime}'

# Find unattached EBS volumes (paying for storage nobody uses)
aws ec2 describe-volumes --filters "Name=status,Values=available" \
  --query 'Volumes[].{ID:VolumeId,Size:Size,Created:CreateTime}'

# Find old snapshots (>90 days, likely forgotten)
aws ec2 describe-snapshots --owner-ids self \
  --query 'Snapshots[?StartTime<=`2026-01-01`].{ID:SnapshotId,Size:VolumeSize,Date:StartTime}'

# Find unused Elastic IPs (charged when not attached)
aws ec2 describe-addresses --query 'Addresses[?AssociationId==null]'
```

**Database waste patterns:**
- RDS instances with Multi-AZ enabled for dev/staging environments (2x cost for no benefit)
- Provisioned IOPS storage when GP3 would suffice (check actual IOPS usage vs provisioned)
- Read replicas that receive <1 query/sec (created "just in case" and forgotten)
- RDS instances sized for peak that happens 2 hours/day (candidate for auto-scaling or scheduled scaling)

**LLM API waste patterns:**

```markdown
## LLM Cost Analysis

| Pattern | Waste Signal | Fix | Estimated Savings |
|---------|-------------|-----|-------------------|
| Wrong model tier | Using GPT-4o/Opus for tasks Haiku/Flash handles fine | Model routing by task complexity | 60-80% on those calls |
| No caching | Same prompts sent repeatedly (FAQ, classification) | Prompt caching / response cache | 30-50% on repeat queries |
| Bloated prompts | System prompts >2000 tokens with unused instructions | Trim to essentials, use progressive disclosure | 15-30% across all calls |
| No streaming | Waiting for full response when streaming would let you fail-fast | Stream + early termination on bad output | 10-20% on long generations |
| Retry storms | Failed calls retried immediately with no backoff | Exponential backoff + circuit breaker | Variable, prevents bill spikes |
| No token budgets | Unbounded max_tokens on every call | Set max_tokens per use case | 10-40% on generation costs |
```

**Model routing decision matrix:**

```
Task complexity: LOW (classification, extraction, yes/no)
  --> Use cheapest model (Haiku, Flash, Groq Llama)
  
Task complexity: MEDIUM (summarization, code generation, structured output)
  --> Use mid-tier (Sonnet, GPT-4o-mini, Gemini Pro)
  
Task complexity: HIGH (reasoning, planning, multi-step analysis)
  --> Use top-tier (Opus, GPT-4o, Gemini Ultra)
  
Task: Embedding generation
  --> Use embedding-specific model (text-embedding-3-small, not a chat model)
```

### Step 3: Right-Sizing Recommendations

For each oversized resource, provide:

```markdown
## Right-Sizing: [Resource Name]

**Current**: [instance type / config] @ $[monthly cost]
**Recommended**: [new instance type / config] @ $[monthly cost]
**Monthly savings**: $[amount] ([percentage]%)
**Risk**: [What could go wrong -- performance degradation, capacity limits]
**Validation**: [How to verify the change is safe -- load test, gradual rollout, monitoring threshold]
**Rollback**: [How to undo if it goes wrong]
```

### Step 4: Commitment Optimization

```markdown
## Reserved / Committed Use Recommendations

### Stable Baseline (always running, predictable)
| Resource | Current (On-Demand) | Recommended | Term | Monthly Savings |
|----------|-------------------|-------------|------|-----------------|
| 3x m6i.xlarge | $XXX/mo | 1yr Reserved (All Upfront) | 12 mo | $XXX (37%) |
| RDS db.r6g.large | $XXX/mo | 1yr Reserved | 12 mo | $XXX (40%) |

### Variable Load (spiky, batch processing)
| Resource | Strategy | Savings vs On-Demand |
|----------|----------|---------------------|
| Batch compute | Spot instances with fallback to on-demand | 60-70% |
| Dev/staging | Scheduled shutdown (nights/weekends) | 65% |
| CI runners | Spot + preemptible with retry | 50-70% |
```

### Step 5: Quick Wins (implement today)

Rank all recommendations by effort vs impact:

```markdown
## Quick Wins (< 1 hour, > $100/mo savings)
1. Delete 12 unattached EBS volumes: -$XXX/mo
2. Release 3 unused Elastic IPs: -$XX/mo
3. Downgrade staging RDS from Multi-AZ: -$XXX/mo
4. Set LLM max_tokens to 1024 for classification calls: -$XXX/mo

## Medium Effort (< 1 week, > $500/mo savings)
1. Right-size 4 EC2 instances: -$X,XXX/mo
2. Implement prompt caching for top 10 LLM calls: -$XXX/mo
3. Move batch jobs to spot instances: -$XXX/mo

## Strategic (> 1 week, > $2000/mo savings)
1. Purchase reserved instances for baseline compute: -$X,XXX/mo
2. Implement model routing for LLM calls: -$X,XXX/mo
3. Migrate from Provisioned to GP3 IOPS on RDS: -$XXX/mo

## Total Estimated Monthly Savings: $XX,XXX
```

## Reporting Format

```markdown
# Cost Optimization Report: [Organization/Project]
**Analysis Period**: [date range]
**Total Monthly Spend**: $XX,XXX
**Identified Savings**: $XX,XXX ([XX]% reduction)
**Implementation Effort**: [X hours/days across N changes]

## Savings by Category
| Category | Current | Optimized | Savings | Effort |
|----------|---------|-----------|---------|--------|
| Compute | $X,XXX | $X,XXX | $X,XXX | 1 day |
| Database | $X,XXX | $X,XXX | $XXX | 2 hours |
| LLM APIs | $X,XXX | $X,XXX | $X,XXX | 1 week |
| Storage | $XXX | $XXX | $XXX | 1 hour |

## Risk Assessment
[Which optimizations are safe (deleting unused resources) vs. risky (right-sizing production compute)]

## Monitoring Plan
[How to verify optimizations didn't degrade performance. Specific metrics to watch for 2 weeks post-change.]
```

## Completion Criteria

- Spend inventory covers >90% of monthly bill
- Every recommendation has: dollar savings, percentage savings, effort estimate, risk level, and rollback plan
- Quick wins are separated from strategic changes
- LLM costs are analyzed at the per-call-pattern level, not just total spend
- No recommendation is "turn it off" without verifying the resource is actually unused
