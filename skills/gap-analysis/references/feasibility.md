# Feasibility Assessment: TELOS Framework

## Overview

TELOS assesses whether a project can realistically be executed across five dimensions. Score each dimension 1-5 and identify blockers before committing resources.

| Dimension | Question |
|-----------|----------|
| **T**echnical | Can we build this with available technology? |
| **E**conomic | Does the financial model work? |
| **L**egal | Are there regulatory barriers? |
| **O**perational | Can the organization support this? |
| **S**chedule | Can we deliver in the required timeframe? |

---

## T - Technical Feasibility

Assess whether required technology and expertise exist.

### Assessment Criteria

| Criterion | Evaluate |
|-----------|----------|
| Technology stack | Available, mature, proven in production |
| Team expertise | Engineers with relevant experience |
| Integrations | APIs available, documented, stable |
| Performance | Latency and throughput requirements achievable |
| Security | Data protection and compliance requirements met |
| Scalability | Clear path to handle 10x growth |

### Risk Classification

| Risk Level | Indicators |
|------------|------------|
| High | Unproven technology, no team expertise, complex integrations with undocumented APIs |
| Medium | New-to-team technology, learning curve 2-4 weeks, some integration complexity |
| Low | Proven stack, experienced team, standard integrations with documented APIs |

### Assessment Questions

1. What technologies are required? Classify each as proven/emerging/experimental.
2. Which team members have built similar systems? Quantify years of experience.
3. List all required integrations. Verify API availability and documentation quality.
4. Define performance requirements: latency p95, throughput, concurrent users.
5. Identify security requirements: encryption, auth, compliance certifications.
6. Project growth: 6-month, 12-month, 24-month scale requirements.

---

## E - Economic Feasibility

Validate the financial model and resource availability.

### Unit Economics Formulas

```
LTV = Average Revenue Per User x Average Customer Lifespan (months)
CAC = Total Acquisition Spend / New Customers Acquired
LTV:CAC Ratio = LTV / CAC
CAC Payback = CAC / Monthly Revenue Per Customer
```

**Target thresholds:**
- LTV:CAC ratio: minimum 3:1
- CAC payback: maximum 12 months

### Project Economics Formulas

```
ROI = (Net Benefit / Total Cost) x 100
Payback Period = Total Investment / Annual Net Cash Flow
NPV = Sum of (Cash Flow / (1 + discount_rate)^year) - Initial Investment
```

### Assessment Criteria

| Criterion | Validate |
|-----------|----------|
| Development cost | Estimated with breakdown by phase |
| Operational cost | Hosting, support, maintenance projected monthly |
| Revenue model | Willingness to pay validated with customer research |
| Break-even | Timeline acceptable given runway |
| ROI | Meets investment threshold (specify target) |
| Funding | Budget approved and available |

### Assessment Questions

1. Total development cost with confidence interval (+/- percentage).
2. Monthly operational costs: hosting, support headcount, maintenance hours.
3. Revenue model: pricing, conversion rates, churn assumptions. Evidence source for each.
4. Break-even timeline in months. Sensitivity to key assumptions.
5. Expected ROI percentage. Compare to company threshold.
6. Funding status: approved, pending approval, requires fundraising.

---

## L - Legal Feasibility

Identify regulatory and legal barriers.

### Regulatory Reference

| Domain | Key Regulations |
|--------|-----------------|
| Data/Privacy | GDPR (EU), CCPA (CA), PIPEDA (Canada), LGPD (Brazil) |
| Healthcare | HIPAA (US), FDA clearance, CE marking (EU) |
| Finance | PCI-DSS (payments), SOX (public companies), PSD2 (EU), AML/KYC |
| Education | FERPA (student data), COPPA (children under 13) |
| Accessibility | ADA, WCAG 2.1 AA, Section 508 |

### Assessment Criteria

| Criterion | Validate |
|-----------|----------|
| Regulations | All applicable regulations identified |
| Licensing | Required licenses and certifications listed |
| IP | Patent search completed, trademark availability confirmed |
| Data privacy | Data collection, storage, and processing compliant |
| Industry compliance | Sector-specific requirements documented |
| Contracts | Existing agreements reviewed for conflicts |

### Assessment Questions

1. List all regulations applicable to product and target markets.
2. Required licenses/certifications with timeline and cost to obtain.
3. Patent landscape: conflicting patents identified? Trademark availability?
4. Data inventory: what data collected, where stored, retention period, deletion process.
5. Existing contracts that constrain what can be built or sold.
6. Compliance documentation required before launch.

---

## O - Operational Feasibility

Assess organizational capacity to support the initiative.

### Readiness Signals

| Status | Indicators |
|--------|------------|
| Ready | Clear ownership assigned, dedicated resources allocated, stakeholder buy-in documented |
| At Risk | Competing priorities, skill gaps requiring training, change resistance identified |
| Not Ready | No ownership, no resources, organizational instability |

### Assessment Criteria

| Criterion | Validate |
|-----------|----------|
| Processes | Required processes defined or can be established |
| Team capacity | Headcount available, not overallocated |
| Support infrastructure | Customer support model defined |
| Training | Training needs scoped with timeline |
| Change management | Impact assessment completed |
| Dependencies | Vendor/partner reliability verified |

### Assessment Questions

1. Named owner with authority and accountability.
2. Team allocation: FTE count, percentage of time, competing priorities.
3. Customer support model: channels, response time SLAs, headcount.
4. Training requirements: topics, hours, delivery method, timeline.
5. Workflow changes: teams affected, migration plan, rollback plan.
6. Third-party dependencies: SLAs, redundancy, exit strategy.

---

## S - Schedule Feasibility

Validate delivery timeline against constraints.

### Risk Factors

- External dependencies: APIs, partners, vendors
- Resource contention with other projects
- Unknown technical complexity
- Regulatory approval timelines
- Holiday and vacation periods

### Estimation Guidance

| Technique | Application |
|-----------|-------------|
| Buffer | Add 30-50% to engineering estimates for novel work, 20% for routine work |
| Critical path | Identify and track items with zero slack |
| Phased delivery | Plan MVP scope to reduce timeline risk |
| Iteration | Reserve 20% of timeline for feedback incorporation |

### Assessment Criteria

| Criterion | Validate |
|-----------|----------|
| Timeline | Development estimate with buffer included |
| Dependencies | Critical path mapped with owners |
| Market timing | Launch window requirements documented |
| Resource availability | Team availability confirmed for duration |
| Contingency | Buffer percentage stated and justified |
| Deadline flexibility | Consequences of delay quantified |

### Assessment Questions

1. Target delivery date. Fixed (external constraint) or flexible (internal goal)?
2. Critical path: longest sequence of dependent tasks with zero slack.
3. Market timing: competitive launches, seasonal factors, regulatory deadlines.
4. Resource availability: confirmed for project duration? Vacation conflicts?
5. Contingency buffer: percentage included, assumptions behind it.
6. Deadline miss impact: revenue loss, competitive disadvantage, contractual penalty.
