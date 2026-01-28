# Prioritization Frameworks

## Table of Contents
- [RICE Scoring](#rice-scoring)
- [ICE Scoring](#ice-scoring)
- [Kano Model](#kano-model)
- [MoSCoW Method](#moscow-method)
- [Framework Selection](#framework-selection)

---

## RICE Scoring

**Formula**: `RICE = (Reach x Impact x Confidence) / Effort`

### Components

| Factor | Definition | Measurement | Scale |
|--------|------------|-------------|-------|
| Reach | Users affected per time period | Actual count | e.g., 500 users/quarter |
| Impact | Effect on individual user | Predefined scale | 3=Massive, 2=High, 1=Medium, 0.5=Low, 0.25=Minimal |
| Confidence | Certainty of estimates | Percentage | 100%=High, 80%=Medium, 50%=Low |
| Effort | Work required | Person-months | Whole or half numbers |

### Calculation Example

```
Feature: Improved onboarding flow
Reach: 500 users/month
Impact: 2 (High)
Confidence: 80%
Effort: 2 person-months

RICE = (500 x 2 x 0.8) / 2 = 400
```

### Interpretation

Compare relative scores between features. Higher score = better ROI. Re-score when estimates change significantly.

---

## ICE Scoring

**Formula**: `ICE = Impact x Confidence x Ease`

### Components

| Factor | Definition | Scale |
|--------|------------|-------|
| Impact | Effect on key metric | 1-10 |
| Confidence | Certainty in estimates | 1-10 |
| Ease | Implementation simplicity | 1-10 |

### Calculation Example

```
Feature: Add social login
Impact: 7 (reduces signup friction)
Confidence: 8 (standard pattern, good data)
Ease: 6 (OAuth integration)

ICE = 7 x 8 x 6 = 336
```

### ICE vs RICE Selection

| Use ICE | Use RICE |
|---------|----------|
| Quick decisions | Strategic roadmap |
| Growth experiments | Large initiatives |
| Small scope items | Quantitative rigor required |
| Early-stage validation | Cross-team prioritization |

---

## Kano Model

Categorize features by customer satisfaction impact.

### Categories

| Category | If Absent | If Present | Priority Action |
|----------|-----------|------------|-----------------|
| Must-Be | Very dissatisfied | Neutral | Implement first - table stakes |
| Performance | Dissatisfied | Satisfied | Invest here - competitive differentiator |
| Delighter | Neutral | Very satisfied | Add selectively - creates loyalty |
| Indifferent | Neutral | Neutral | Skip or deprioritize |
| Reverse | Satisfied | Dissatisfied | Do not implement |

### Visual Model

```
Satisfaction
     ^
     |      / Delighters
     |     /
     |    /
     |   /------ Performance
     |  /
-----+-----------------------> Functionality
     |   ======= Must-Be
     |
     v
```

### Survey Method

Ask two questions per feature:

1. **Functional**: "If [feature] were present, how would you feel?"
2. **Dysfunctional**: "If [feature] were absent, how would you feel?"

Response options: Like, Expect, Neutral, Tolerate, Dislike

Cross-reference answers using Kano evaluation table to categorize.

### Feature Evolution

Features shift categories over time: **Delighter** -> **Performance** -> **Must-Be**

Example: Mobile apps were delighters (2010), now must-be (baseline expectation).

---

## MoSCoW Method

Categorize requirements by necessity for a specific release.

### Categories

| Category | Definition | Effort Allocation |
|----------|------------|-------------------|
| Must Have | Non-negotiable for launch | <=60% of capacity |
| Should Have | Important, workaround exists | ~20% of capacity |
| Could Have | Desirable if resources permit | Remaining capacity |
| Won't Have | Explicitly deferred | 0% - goes to backlog |

### Application Rules

1. Must Haves MUST NOT exceed 60% of total effort
2. Should Haves fill ~20% of effort budget
3. Could Haves use remaining capacity as buffer
4. Won't Haves are documented, not forgotten

### Example: E-commerce MVP

| Must Have | Should Have | Could Have | Won't Have |
|-----------|-------------|------------|------------|
| Product catalog | User accounts | Wishlist | Loyalty program |
| Shopping cart | Order history | Product reviews | Gift cards |
| Checkout flow | Email confirmations | Related products | International shipping |
| Payment processing | | | |

---

## Framework Selection

### Decision Matrix

| Situation | Framework |
|-----------|-----------|
| Comparing many features quantitatively | RICE |
| Quick growth experiment prioritization | ICE |
| Understanding customer value perception | Kano |
| Scoping a specific release | MoSCoW |
| Strategic roadmap planning | RICE + Kano |
| Sprint planning | MoSCoW |

### Combined Workflow

1. **Kano**: Categorize features by customer value type
2. **RICE**: Score and rank opportunities quantitatively
3. **MoSCoW**: Scope specific releases from ranked backlog
4. **Re-evaluate**: Update scores when context changes significantly
