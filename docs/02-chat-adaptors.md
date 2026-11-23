User: >>`model can learn latent semantics such as:
- this sequence resembles a recurring liquidity squeeze
- this spending pattern is seasonal and goal-linked
- this cluster is associated with a project-like consumption pattern
- this is a likely anomaly relative to prior behavior
- this user’s behavior suggests a different decision regime than their baseline pattern`

>>`system turns scattered spending history into a project memory with explainable context.`

>>`core product thesis:
- reconstruct implicit decisions from transactional behavior
- turn them into explicit, trustworthy financial semantics
- let users and models collaborate around goals, constraints, and strategies`

List here 1-2 data examples for each one of the above features.

GitHub Copilot: ## 1) Latent semantic: recurring liquidity squeeze

**Example 1 — Payday-to-payday squeeze**

```json
{
  "household_id": "H-22",
  "window": "2026-09-01/2026-11-01",
  "sequence": [
    {"date": "2026-09-28", "event": "credit_card_minimum_payment", "amount": 420},
    {"date": "2026-10-01", "event": "salary_received", "amount": 3800},
    {"date": "2026-10-07", "event": "checking_balance_low", "amount": 140},
    {"date": "2026-10-15", "event": "payday_advance", "amount": 300},
    {"date": "2026-10-28", "event": "credit_card_minimum_payment", "amount": 420}
  ],
  "latent_semantics": {
    "pattern": "recurring_liquidity_squeeze",
    "confidence": 0.87,
    "typical_period_days": 30,
    "signals": ["post-payday spending", "pre-payday low balance", "repeated minimum payment"]
  }
}
```

**Example 2 — Fixed-cost timing squeeze**

```json
{
  "household_id": "H-22",
  "sequence": [
    {"event": "rent", "amount": 2400},
    {"event": "insurance", "amount": 680},
    {"event": "car_payment", "amount": 520},
    {"event": "checking_buffer_below_target", "amount": -120}
  ],
  "latent_semantics": {
    "pattern": "recurring_liquidity_squeeze",
    "confidence": 0.79,
    "cause": "fixed_obligations_cluster_before_next_income",
    "affected_window": "first_7_days_of_month"
  }
}
```

---

## 2) Seasonal and goal-linked spending

**Example 1 — Travel goal**

```json
{
  "sequence": [
    {"month": "2025-11", "event": "flight_search"},
    {"month": "2025-12", "event": "hotel_deposit", "amount": 900},
    {"month": "2026-01", "event": "travel_insurance", "amount": 240},
    {"month": "2026-01", "event": "travel_booking", "amount": 1200}
  ],
  "latent_semantics": {
    "pattern": "seasonal_goal_linked_spending",
    "goal": "family_winter_travel",
    "season": "winter",
    "confidence": 0.84,
    "recurrence": "annual"
  }
}
```

**Example 2 — Home-maintenance season**

```json
{
  "sequence": [
    {"month": "2026-03", "merchant_category": "gardening", "amount": 280},
    {"month": "2026-04", "merchant_category": "hardware", "amount": 520},
    {"month": "2026-05", "merchant_category": "landscaping", "amount": 1400}
  ],
  "latent_semantics": {
    "pattern": "seasonal_project_spending",
    "likely_project": "spring_home_maintenance",
    "confidence": 0.81
  }
}
```

---

## 3) Project-like consumption cluster

**Example 1 — Home renovation project**

```json
{
  "project_id": "P-91",
  "transaction_sequence": [
    {"event": "paint_purchase", "amount": 180},
    {"event": "hardware_purchase", "amount": 310},
    {"event": "tool_rental", "amount": 220},
    {"event": "contractor_payment", "amount": 2400},
    {"event": "delivery_fee", "amount": 95}
  ],
  "latent_semantics": {
    "pattern": "project_like_consumption",
    "likely_project": "home_maintenance",
    "budget_status": "at_risk",
    "confidence": 0.91
  }
}
```

**Example 2 — Education/project cluster**

```json
{
  "transaction_sequence": [
    {"event": "course_fee", "amount": 1400},
    {"event": "textbook_purchase", "amount": 220},
    {"event": "software_subscription", "amount": 49},
    {"event": "conference_ticket", "amount": 750}
  ],
  "latent_semantics": {
    "pattern": "project_like_consumption",
    "likely_project": "professional_development",
    "goal_alignment": "career_advancement",
    "confidence": 0.76
  }
}
```

---

## 4) Likely anomaly relative to prior behavior

**Example 1 — Dining spend spike**

```json
{
  "user_id": "U-1042",
  "category": "dining",
  "prior_90_day_average": 320,
  "current_month": 710,
  "sequence": [
    {"event": "restaurant", "amount": 180},
    {"event": "delivery", "amount": 95},
    {"event": "restaurant", "amount": 260}
  ],
  "latent_semantics": {
    "pattern": "behavioral_anomaly",
    "deviation": 122,
    "confidence": 0.88,
    "possible_explanation": "temporary_life_event",
    "requires_confirmation": true
  }
}
```

**Example 2 — Unusual savings reduction**

```json
{
  "baseline_monthly_savings": 800,
  "current_monthly_savings": 100,
  "cause_signals": ["car_repair", "insurance_renewal", "withdrawal_from_savings"],
  "latent_semantics": {
    "pattern": "anomaly",
    "risk": "cash_buffer_reduction",
    "confidence": 0.83,
    "should_not_infer": ["impulsive_behavior"]
  }
}
```

---

## 5) Different decision regime from baseline

**Example 1 — Defensive cash-conservation mode**

```json
{
  "baseline_regime": "growth_ready",
  "observed_signals": {
    "discretionary_spend_change": -38,
    "credit_utilization_change": 24,
    "savings_transfer_change": -60,
    "income_variance": 0.31
  },
  "latent_semantics": {
    "current_regime": "cash_conservation",
    "departure_from_baseline": 0.79,
    "confidence": 0.84,
    "recommended_treatment": "pause_risky_strategy_changes"
  }
}
```

**Example 2 — Debt-reduction acceleration**

```json
{
  "baseline_regime": "balanced",
  "observed_signals": {
    "extra_debt_payments": 3,
    "debit_card_spending_change": -18,
    "checking_buffer_change": -12
  },
  "latent_semantics": {
    "current_regime": "debt_reduction_acceleration",
    "confidence": 0.74,
    "consistent_with_goal": "credit_card_debt_payoff",
    "requires_review": ["cash_buffer_below_target"]
  }
}
```

---

## 6) Project memory with explainable context

**Example 1 — Complete home-project memory**

```json
{
  "project_id": "P-91",
  "project_name": "Home maintenance",
  "status": "active",
  "evidence": [
    {"event": "paint_purchase", "amount": 180, "date": "2026-09-03"},
    {"event": "hardware", "amount": 310, "date": "2026-09-08"},
    {"event": "contractor_payment", "amount": 2400, "date": "2026-09-20"}
  ],
  "context": {
    "goal": "maintain_home",
    "budget": 5000,
    "spent": 2890,
    "remaining": 2110,
    "constraint": "preserve_emergency_fund"
  },
  "competing_context": ["family_travel", "credit_card_pressure"],
  "explanation": "The recurring cluster indicates an active home-maintenance project. It competes with travel and household liquidity.",
  "correction_allowed": true
}
```

**Example 2 — Travel project memory**

```json
{
  "project_id": "P-117",
  "project_name": "Summer family travel",
  "status": "planning",
  "evidence": [
    {"event": "flight_booking", "amount": 1600},
    {"event": "hotel_deposit", "amount": 900},
    {"event": "travel_insurance", "amount": 280}
  ],
  "context": {
    "goal": "summer_travel",
    "budget": 4500,
    "spent": 2780,
    "remaining": 1720,
    "season": "summer"
  },
  "explanation": "Repeated bookings over 6 weeks indicate a deliberate travel project rather than isolated discretionary spending.",
  "user_edit_options": ["confirm_project", "merge_project", "reject_inference"]
}
```

---

## 7) Core thesis: reconstruct implicit decisions

**Example 1 — Dining decision becomes explicit**

```json
{
  "observed_behavior": [
    {"event": "takeout", "amount": 45, "weekday": "Friday"},
    {"event": "takeout", "amount": 38, "weekday": "Friday"},
    {"event": "takeout", "amount": 52, "weekday": "Friday"}
  ],
  "decision_hypothesis": {
    "implicit_decision": "use_delivery_as_weekly_end_of_week_recovery",
    "confidence": 0.68,
    "goal_effect": "small_reduction_in_savings_progress"
  },
  "explicit_user_review": {
    "prompt": "Is Friday dining part of a household routine?",
    "options": ["routine", "temporary_habit", "project_or_event"]
  }
}
```

**Example 2 — Extra debt payment becomes explicit**

```json
{
  "observed_behavior": [
    {"event": "minimum_card_payment", "amount": 420},
    {"event": "extra_card_payment", "amount": 800},
    {"event": "extra_card_payment", "amount": 800}
  ],
  "decision_hypothesis": {
    "implicit_decision": "prioritize_credit_card_repayment",
    "confidence": 0.89,
    "tradeoff": "less_investment_and_emergency_fund_contribution"
  }
}
```

---

## 8) Core thesis: explicit, trustworthy financial semantics

**Example 1 — Semantic transaction**

```json
{
  "transaction_id": "T-1091",
  "raw": {"merchant": "Northline Hardware", "amount": 310},
  "explicit_semantics": {
    "category": "home_maintenance",
    "likely_project": "P-91",
    "goal": "maintain_home",
    "constraint": "protect_emergency_reserve",
    "confidence": 0.91,
    "evidence": ["repeat_merchant", "hardware_category", "paint_purchase_sequence"]
  }
}
```

**Example 2 — Semantic risk state**

```json
{
  "household_id": "H-22",
  "as_of": "2026-11-18",
  "explicit_semantics": {
    "state": "liquidity_pressure",
    "severity": "moderate",
    "evidence": ["cash_buffer_below_target", "three_minimum_payments", "income_volatility"],
    "unconfirmed": ["credit_card_rate_change"],
    "recommended_action_type": "protect_liquidity_before_growth"
  }
}
```

---

## 9) Core thesis: user/model collaboration around goals, constraints, and strategies

**Example 1 — Goal correction**

```json
{
  "inferred_goal": "home_maintenance",
  "predicted_constraint": "monthly_budget_cap",
  "suggested_strategy": "phase_project_across_two_months",
  "user_override": {
    "confirmed_goal": "home_maintenance",
    "change_constraint": "extend_timeline",
    "selected_strategy": "defer_discretionary_travel"
  }
}
```

**Example 2 — Strategy review**

```json
{
  "strategy_candidates": [
    "debt_first",
    "mixed_50_50",
    "preserve_emergency_reserve"
  ],
  "model_output": {
    "preferred": "debt_first",
    "reason": "high_credit_apr_and_below_target_cash_buffer"
  },
  "user_input": {
    "cash_buffer_preference": "retain_additional_3000",
    "risk_preference": "conservative"
  },
  "result": {
    "selected_strategy": "debt_first_with_cash_buffer_guardrail",
    "requires_reconfirmation": true
  }
}
```

Each example should retain uncertainty and expose supporting evidence. The model should infer hypotheses, but the user should be able to confirm, correct, or reject them.

User: Using the above examples, give evidence on how the TFM model will learn and extract explicit spending patterns, different decision regimes, associate with goal/ projects etc. 

GitHub Copilot: ## How TFM learns the patterns

A TFM should learn from three complementary evidence streams:

1. **Transactions** — amounts, category, merchant, account, timing
2. **Context** — household, goals, budget, constraints, debt, income
3. **Feedback** — accepted or corrected goal, project, strategy, and anomaly labels

The TFM does not infer all of this from raw text alone. It needs structured event sequences and weak or confirmed labels.

---

## 1. Liquidity-squeeze patterns

### Training evidence

The TFM sees sequences such as:

```text
salary_received
→ credit_card_minimum_payment
→ checking_balance_low
→ payday_advance
→ credit_card_minimum_payment
→ repeated_payday
```

The model learns associations between:

- event timing
- amount relative to income
- fixed-cost density
- low balances
- repeated monthly timing
- debt-payment behavior

### Explicit extraction

It can produce:

```json
{
  "latent_class": "recurring_liquidity_squeeze",
  "confidence": 0.87,
  "evidence": [
    "balance low in the week before four of five paydays",
    "credit-card payment occurs after every identified payday",
    "fixed obligations consume more than 70% of next income"
  ]
}
```

### Evidence quality

The strongest evidence is temporal recurrence. An isolated low balance should be an anomaly or a weak signal, not a liquidity-squeeze classification.

The TFM must verify:
- persistence across several cycles
- relationship to income timing
- amount and severity
- whether the pattern is caused by obligations or discretionary spending

---

## 2. Seasonal and goal-linked spending

### Training evidence

Examples must include sequences across many months:

```text
travel_search
→ flight_booking
→ hotel_deposit
→ insurance
→ reduced dining spend
→ travel_completed
```

The model also receives goal context:

```json
{
  "goal": "family_winter_travel",
  "target_date": "2027-01-15",
  "season": "winter",
  "budget": 4500
}
```

### What it learns

The TFM learns that:
- certain merchants belong to the travel context
- spending repeats around the same season
- several categories form one decision thread
- travel spending is often linked to a declared or inferred goal
- goal-linked purchases differ from a single luxury purchase

### Explicit extraction

```json
{
  "pattern": "seasonal_goal_linked_spending",
  "goal": "family_winter_travel",
  "season": "winter",
  "confidence": 0.84,
  "evidence": [
    "repeated winter bookings",
    "same merchant categories across prior years",
    "goal-linked planning events",
    "budgeted spending began 90 days before travel"
  ]
}
```

Seasonality alone is insufficient. The TFM should require both periodic behavior and semantic linkage to goals or projects.

---

## 3. Project-like consumption clusters

### Training evidence

A home-project sequence may contain:

```text
paint_purchase
→ hardware_purchase
→ tool_rental
→ contractor_payment
→ delivery_fee
```

The model should learn a cluster because the events:
- share a household
- occur within a bounded time window
- contain related merchants or categories
- exceed expected discretionary spending
- affect an active goal or budget
- are recurrently followed by progress or completion activity

### Explicit extraction

```json
{
  "likely_project": "home_maintenance",
  "confidence": 0.91,
  "cluster_window": "2026-09-03/2026-09-20",
  "evidence": [
    "paint and hardware purchases occurred within five days",
    "three tools and delivery expenses support the same project",
    "contractor payment belongs to the same household goal",
    "project spending exceeds the prior monthly baseline"
  ]
}
```

### Goal association

The TFM should not infer that every hardware purchase means a project. Household context and multiple supporting events are more reliable than one transaction.

Example evidence dimensions:
- merchant similarity
- category continuity
- temporal proximity
- amount continuity
- goal keyword or metadata
- user confirmation
- budget impact

---

## 4. Behavioral anomalies

### Training evidence

The TFM compares current behavior against a household-specific baseline:

```text
prior 90-day dining average: $320
current-month dining spend: $710
```

But the model should not classify a deviation purely by amount. It should inspect:
- one-time event versus recurrence
- seasonal context
- income and cash-flow changes
- household member activity
- travel or project context
- declarations from the user

### Explicit extraction

```json
{
  "pattern": "behavioral_anomaly",
  "category": "dining",
  "deviation_percent": 122,
  "confidence": 0.88,
  "possible_causes": [
    "temporary_life_event",
    "travel",
    "changed work schedule"
  ],
  "must_request_confirmation": true
}
```

The system must not automatically conclude that the user is careless or irresponsible. A robust TFM should identify uncertainty and present plausible explanations.

---

## 5. Different decision regimes

The TFM can learn a decision regime as a latent state over a time window.

### Baseline state

```json
{
  "baseline_regime": "balanced",
  "features": {
    "discretionary_spend_trend": "stable",
    "savings_transfer": "consistent",
    "debt_utilization": "moderate"
  }
}
```

### New state

```json
{
  "current_regime": "cash_conservation",
  "features": {
    "discretionary_spend_change": -38,
    "credit_utilization_change": 24,
    "savings_transfer_change": -60,
    "income_variance": 0.31
  }
}
```

### Learning process

The TFM learns regime transitions from sequences such as:

```text
income_variation
→ lower_discretionary_spend
→ credit_utilization_increase
→ lower_savings_transfer
→ reduced_project_activity
```

It can then learn that such a sequence resembles a defensive or cash-conservation regime rather than normal balanced behavior.

### User-safe extraction

```json
{
  "baseline_regime": "growth_ready",
  "current_regime": "cash_conservation",
  "confidence": 0.84,
  "evidence": [
    "three-month savings decline",
    "discretionary spending reduced by 38%",
    "credit utilization increased"
  ],
  "strategy_implication": "prefer liquidity-protective strategies"
}
```

The regime should be reversible. When income stabilizes or debt utilization declines, the TFM should learn a transition back toward the baseline regime.

---

## 6. Explicit project memory

### Stored memory object

```json
{
  "project_id": "P-91",
  "name": "Home maintenance",
  "status": "active",
  "evidence": [
    "paint purchase",
    "hardware purchase",
    "tool rental",
    "contractor payment"
  ],
  "budget": 5000,
  "spent": 2890,
  "constraint": "preserve_emergency_reserve"
}
```

### How TFM creates it

The TFM can transform a sequence:

```text
hardware
→ paint
→ tools
→ contractor
→ household maintenance goal
→ budget pressure
```

into a typed memory:

```text
project_id P-91
goal_id G-02
household_id H-22
constraint liquidity
status active
```

A useful TFM pipeline is:

1. encode the transaction sequence
2. infer candidate project clusters
3. attach goal and budget context
4. calculate confidence
5. present the proposed project memory
6. allow user confirmation or correction
7. store corrected labels as supervised or reinforcement feedback

This creates a loop rather than one-way inference.

---

## 7. Explicit spending semantics

The TFM should convert a raw event:

```json
{
  "merchant": "Northline Hardware",
  "amount": 310
}
```

into a typed semantic event:

```json
{
  "category": "home_maintenance",
  "project": "P-91",
  "goal": "maintain_home",
  "constraint": "preserve_emergency_reserve",
  "confidence": 0.91
}
```

### Evidence trace

The trace might include:

- merchant category
- prior hardware purchases
- related paint purchases
- temporal cluster
- active household project
- declared project budget
- user correction history

This makes the result explainable rather than merely predictive.

---

## 8. User-model collaboration

The TFM should never treat its extraction as ground truth.

For each latent result, it should expose:

- predicted meaning
- confidence
- supporting evidence
- possible alternatives
- suggested user action
- correction mechanism

For example:

```json
{
  "inferred_project": "home_maintenance",
  "confidence": 0.91,
  "alternatives": ["home_office_setup", "property_upgrade"],
  "prompt": "Is this purchase part of your home-maintenance project?",
  "actions": ["confirm", "correct", "reject"]
}
```

Every confirmation or correction becomes a future training signal.

---

## 9. Training architecture

A practical learning stack is:

```text
transaction sequence
        ↓
semantic tokenization
        ↓
context fusion: user, household, goal, project, budget
        ↓
TFM latent representation
        ↓
task adapters:
  - liquidity pressure
  - seasonality
  - project clustering
  - anomaly detection
  - decision regime
        ↓
typed evidence objects
        ↓
strategy and recommendation engine
        ↓
user confirmation / correction
        ↓
training memory update
```

## Critical evidence requirements

A TFM should learn patterns reliably only when the dataset includes:

- more than one household
- multiple months of sequences
- recurring and isolated events
- explicit and inferred goals
- accepted and rejected recommendations
- positive and negative project matches
- confirmed and corrected anomalies
- income volatility
- household and individual context
- balance and debt state

Without this data, the TFM will tend to overfit obvious category patterns and miss the higher-level decision meaning.

## Bottom line

The TFM learns explicit spending patterns by combining **temporal recurrence + merchant/category semantics + household context + goal association + user feedback**.

It does not extract projects or decision regimes from transaction labels alone. It learns them by identifying which events belong to a stable sequence, why they occurred together, and how they relate to goals and constraints.

User: >>`task adapters:
  - liquidity pressure
  - seasonality
  - project clustering
  - anomaly detection
  - decision regime`
>> `typed evidence objects`

Provide test cases on how to test that model extracted the explicit spending patterns correctly

GitHub Copilot: ## Test strategy

Test each adapter on three levels:

1. **Schema and required fields**
2. **Prediction correctness**
3. **Evidence correctness**
4. **Counterfactual negatives**
5. **User-correction learning**

A prediction is not correct merely because the label is correct; the supporting evidence must also be relevant and the output must not leak sensitive or unsupported claims.

---

## Shared hard gates

Every test should fail immediately if:

- output is not valid JSON
- required fields are missing
- enum values are invalid
- evidence references nonexistent events
- confidence is outside $0$–$1$
- an adapter claims certainty for a genuinely uncertain pattern

```json
{
  "required_fields": [
    "adapter",
    "predicted_type",
    "confidence",
    "evidence_event_ids",
    "requires_user_confirmation",
    "contraindications"
  ]
}
```

---

## 1. Liquidity-pressure adapter

### Positive test

```json
{
  "test_id": "LIQ-001",
  "input": [
    {"id": "E1", "type": "salary_received", "amount": 3800, "day": 1},
    {"id": "E2", "type": "rent", "amount": 2400, "day": 2},
    {"id": "E3", "type": "checking_balance_low", "amount": 150, "day": 7},
    {"id": "E4", "type": "minimum_debt_payment", "amount": 420, "day": 28},
    {"id": "E5", "type": "checking_balance_low", "amount": 110, "day": 29}
  ],
  "expected": {
    "predicted_type": "recurring_liquidity_squeeze",
    "min_confidence": 0.75,
    "required_evidence": ["E2", "E3", "E4", "E5"],
    "requires_user_confirmation": true
  }
}
```

**Assert:**
- Pattern spans at least two repeated cycles.
- Fixed obligations precede low-balance windows.
- Debt burden is included in evidence, but is not asserted as the sole cause.

### Negative test

```json
{
  "test_id": "LIQ-NEG-001",
  "input": [
    {"id": "E1", "type": "large_one_time_purchase", "amount": 2200},
    {"id": "E2", "type": "checking_balance_low", "amount": 180}
  ],
  "expected": {
    "predicted_type": "isolated_liquidity_event",
    "must_not_predict": "recurring_liquidity_squeeze"
  }
}
```

---

## 2. Seasonality adapter

### Positive test

```json
{
  "test_id": "SEA-001",
  "input": [
    {"id": "T1", "month": "2025-11", "category": "travel_search"},
    {"id": "T2", "month": "2025-12", "category": "hotel_deposit", "amount": 900},
    {"id": "T3", "month": "2026-11", "category": "flight_booking"},
    {"id": "T4", "month": "2026-12", "category": "hotel_deposit", "amount": 950}
  ],
  "goal_context": {"goal": "family_winter_travel", "target_month": "12"},
  "expected": {
    "predicted_type": "seasonal_goal_linked_spending",
    "required_evidence": ["T1", "T2", "T3", "T4"],
    "must_include": ["goal_id", "season", "confidence"]
  }
}
```

**Assert:**
- The model finds repeating travel behavior across at least two annual cycles.
- It associates the pattern with the travel goal.
- It does not call it seasonal merely because a single December transaction exists.

### Negative test

```json
{
  "test_id": "SEA-NEG-001",
  "input": [
    {"id": "X1", "month": "2026-12", "category": "dining", "amount": 240},
    {"id": "X2", "month": "2026-12", "category": "gifts", "amount": 500}
  ],
  "expected": {
    "must_not_predict": "seasonal_goal_linked_spending"
  }
}
```

---

## 3. Project-clustering adapter

### Positive test

```json
{
  "test_id": "PROJ-001",
  "input": [
    {"id": "P1", "category": "paint", "amount": 180, "day": 3},
    {"id": "P2", "category": "hardware", "amount": 310, "day": 5},
    {"id": "P3", "category": "tool_rental", "amount": 220, "day": 8},
    {"id": "P4", "category": "contractor", "amount": 2400, "day": 18},
    {"id": "P5", "category": "delivery", "amount": 95, "day": 19}
  ],
  "goal_context": {"goal_id": "G-02", "goal": "home_maintenance"},
  "expected": {
    "predicted_type": "project_like_consumption",
    "predicted_project": "home_maintenance",
    "required_evidence": ["P1", "P2", "P3", "P4", "P5"],
    "requires_user_confirmation": true
  }
}
```

**Assert:**
- Related categories are grouped within a bounded window.
- Each evidence event supports the same project hypothesis.
- Project budget and goal context can be attached.

### Splitter test

```json
{
  "test_id": "PROJ-002",
  "input": [
    {"id": "A1", "category": "paint", "day": 1},
    {"id": "A2", "category": "paint", "day": 90},
    {"id": "A3", "category": "hardware", "day": 95}
  ],
  "expected": {
    "must_not_merge_as_one_project": true,
    "require_separate_windows": true
  }
}
```

---

## 4. Anomaly adapter

### Positive test

```json
{
  "test_id": "ANOM-001",
  "baseline": {
    "dining_monthly_average": 320,
    "standard_deviation": 55
  },
  "input": [
    {"id": "D1", "category": "dining", "amount": 180},
    {"id": "D2", "category": "dining", "amount": 260},
    {"id": "D3", "category": "delivery", "amount": 95}
  ],
  "expected": {
    "predicted_type": "behavioral_anomaly",
    "deviation_percent_min": 100,
    "requires_confirmation": true,
    "must_not_assert_cause": true
  }
}
```

**Assert:**
- The result is household-specific.
- Confidence reflects deviation magnitude and sample quality.
- Possible causes are hypotheses, not conclusions.
- The user can reject “overspending” as the explanation.

### Negative test

```json
{
  "test_id": "ANOM-NEG-001",
  "baseline": {
    "dining_monthly_average": 320,
    "standard_deviation": 55
  },
  "input": [
    {"id": "D1", "category": "dining", "amount": 335}
  ],
  "expected": {
    "must_not_predict": "behavioral_anomaly"
  }
}
```

---

## 5. Decision-regime adapter

### Positive transition test

```json
{
  "test_id": "REGIME-001",
  "baseline": "growth_ready",
  "input": [
    {"id": "R1", "metric": "discretionary_spend_change", "value": -38},
    {"id": "R2", "metric": "credit_utilization_change", "value": 24},
    {"id": "R3", "metric": "savings_transfer_change", "value": -60},
    {"id": "R4", "metric": "income_variance", "value": 31}
  ],
  "expected": {
    "predicted_type": "cash_conservation",
    "required_evidence": ["R1", "R2", "R3", "R4"],
    "must_not_infer_regime_permanence": true
  }
}
```

**Assert:**
- A window of evidence changes the regime.
- Income volatility is considered.
- The system returns a transition, not a permanent personality label.
- It should provide a timestamp or window for the regime.

### Negative test

```json
{
  "test_id": "REGIME-NEG-001",
  "baseline": "balanced",
  "input": [
    {"id": "R1", "metric": "single_transaction", "category": "dining", "amount": 180}
  ],
  "expected": {
    "must_not_predict": "decision_regime_change"
  }
}
```

---

## Typed evidence-object tests

### Required evidence schema

```json
{
  "evidence_id": "EV-001",
  "source_event_id": "P1",
  "event_type": "merchant_transaction",
  "contribution": "supports_project",
  "weight": 0.82,
  "timestamp": "2026-09-03",
  "context": ["home_maintenance", "household_H22"]
}
```

### Evidence tests

- A project inference must include at least two event IDs.
- Every cited evidence ID must exist in the source sequence.
- Evidence must not rely on removed or redacted transactions.
- Evidence objects must show whether they support or contradict the prediction.
- Confidence must be testable as a score or calibrated probability, not a vague label.
- Two models should produce equivalent evidence for identical inputs.

---

## Cross-adapter tests

### User correction

```json
{
  "test_id": "USER-CORRECT-001",
  "model_output": {
    "inferred_project": "home_maintenance",
    "confidence": 0.91
  },
  "user_correction": {
    "corrected_project": "home_office_setup",
    "rejection_reason": "purchase_sequence_belongs_to_other_project"
  },
  "expected": {
    "project_memory": "home_office_setup",
    "inference_confidence_changed": true,
    "feedback_event_created": true
  }
}
```

### Ambiguous sequence

```json
{
  "test_id": "AMBIG-001",
  "input": [
    {"category": "hardware", "amount": 310},
    {"category": "paint", "amount": 180}
  ],
  "expected": {
    "predicted_type": "project_candidate",
    "confidence_max": 0.69,
    "requires_user_confirmation": true,
    "must_not_assert_complete_project": true
  }
}
```

---

## Suggested evaluation metrics

| Adapter | Primary metric | Also measure |
|---|---|---|
| Liquidity pressure | Precision and recall | Precision@top evidence, false alarm rate |
| Seasonality | F1 | Year-to-year recurrence recall |
| Project clustering | Cluster purity | Window segmentation, evidence precision |
| Anomaly detection | Precision at k | False positive rate and severity calibration |
| Decision regime | Macro-F1 | Transition timing and recovery recall |

Minimum evidence checks:

- **Evidence recall:** percentage of supporting source events returned
- **Evidence precision:** percentage of returned evidence that genuinely supports the label
- **Contradiction detection:** whether opposing evidence is acknowledged
- **Calibration error:** whether a $0.80$ confidence prediction is correct approximately $80\%$ of the time
- **User review rate:** percentage of uncertain predictions that trigger confirmation
- **Correction rate:** percentage of confirmed predictions later reversed

## Recommended test-suite structure

```text
tests/
  adapters/
    liquidity_pressure/
    seasonality/
    project_clustering/
    anomaly/
    decision_regime/
  gold/
    positive_examples.json
    negative_examples.json
    ambiguous_examples.json
  evidence/
    valid_evidence_objects.json
    dangling_references.json
    contradictory_evidence.json
  feedback/
    confirmations.json
    corrections.json
    rejected_predictions.json
```

The key test principle is that a correct label is insufficient. The suite should prove that the TFM learned the pattern from appropriate temporal evidence, associated it with the correct goal or project context, acknowledged uncertainty, and produced an auditable evidence trail.

User:  How to these multiple adaptors interact at runtime? 
 Are they orchestrated by the TFM model, by the inference engine?
 Explain how and when are they invoked.

GitHub Copilot: ## Runtime ownership

The TFM should own **encoding and state**, not independently run business logic. The inference engine should own **adapter routing, evidence fusion, policy, and orchestration**.

```text
Financial events
      ↓
TFM / sequence encoder
      ↓
Contextual latent representation
      ↓
Inference engine: routing + policies
      ↓
Adapters: liquidity, seasonality, project, anomaly, regime
      ↓
Evidence fusion
      ↓
Typed DecisionContext / Recommendation
```

This boundary is important:
- TFM asks: “What does this sequence mean?”
- Inference engine asks: “Which interpretations are relevant, reliable, and permissible?”
- Adapters ask: “What evidence supports this one semantic task?”
- Policies decide: “What may be shown or acted upon?”

---

## 1. TFM is always upstream

The TFM should run on a newly observed event window and produce a contextual representation.

```json
{
  "sequence_id": "SEQ-H22-202611",
  "household_id": "H-22",
  "window": {
    "start": "2026-10-01",
    "end": "2026-11-23"
  },
  "latent_state": {
    "liquidity_embedding": [0.12, -0.44, 0.77],
    "spending_embedding": [-0.22, 0.31, 0.18],
    "goal_context_embedding": [0.56, 0.04, -0.91]
  },
  "token_summary": {
    "salary_events": 2,
    "low_balance_events": 4,
    "project_like_events": 6,
    "anomalous_events": 1
  }
}
```

The TFM does not need to decide that a sequence is a liquidity squeeze. Its representation should provide evidence for downstream adapters.

---

## 2. Inference engine determines which adapters run

The inference engine is the runtime orchestrator.

It routes work based on:

- event type
- context availability
- minimum history length
- data freshness
- active goals
- active strategies
- confidence thresholds
- cost and latency
- household permissions
- whether an adapter has run recently

Example route plan:

```json
{
  "sequence_id": "SEQ-H22-202611",
  "routes": [
    {"adapter": "liquidity_pressure", "priority": 1, "run": true},
    {"adapter": "project_clustering", "priority": 2, "run": true},
    {"adapter": "seasonality", "priority": 3, "run": true},
    {"adapter": "anomaly", "priority": 4, "run": true},
    {"adapter": "decision_regime", "priority": 5, "run": true}
  ]
}
```

A transaction-only event may invoke only anomaly and enrichment. An income or balance event may invoke liquidity. A cluster of goal-linked purchases may invoke project clustering and seasonality.

---

## 3. Parallel, then dependent stages

Adapters are not simply called in a fixed chain. There are usually three stages.

### Stage A: independent adapters

These can run concurrently because they have the TFM representation and no final shared decision:

- liquidity pressure
- seasonality
- project clustering
- anomaly detection
- decision regime

Example:

```text
TFM representation
  ├── liquidity adapter
  ├── seasonality adapter
  ├── project adapter
  ├── anomaly adapter
  └── decision-regime adapter
```

Each adapter should return a partial typed object:

```json
{
  "adapter": "project_clustering",
  "status": "candidate",
  "prediction": "home_maintenance",
  "confidence": 0.91,
  "evidence_event_ids": ["P1", "P2", "P3", "P4"],
  "requires_confirmation": true
}
```

### Stage B: fusion

The inference engine merges overlapping interpretations.

Example:

- project adapter predicts home maintenance
- seasonality adapter predicts a spring maintenance cycle
- anomaly adapter predicts elevated spending
- liquidity adapter predicts moderate cash pressure
- decision-regime adapter predicts cash conservation

A naïve union would say five unrelated things. Fusion should produce one interpretation:

```json
{
  "typed_semantics": [
    {
      "type": "active_project",
      "value": "home_maintenance",
      "confidence": 0.91
    },
    {
      "type": "seasonal_pattern",
      "value": "spring_maintenance",
      "confidence": 0.81
    },
    {
      "type": "liquidity_pressure",
      "value": "moderate",
      "confidence": 0.76
    }
  ],
  "dominant_decision_regime": "cash_conservation"
}
```

Fusion should preserve contradiction instead of forcing consensus.

### Stage C: policy and strategy

The inference engine then:

- removes unsupported conclusions
- checks guardrails
- ranks compatible interpretations
- passes context to the strategy adapter
- decides whether to request user confirmation
- decides whether to generate a recommendation

For example:

```text
active project = home maintenance
liquidity state = moderate pressure
goal = preserve emergency fund
constraint = minimum $5,000 cash buffer
→ strategy candidate = phase the project
→ confirmation = required
```

---

## 4. When adapters should run

| Trigger | Adapters |
|---|---|
| New transaction | Anomaly, semantic enrichment, project, relevant seasonality |
| Balance or debt changed | Liquidity, decision regime |
| New income or missed transfer | Liquidity, anomaly, decision regime |
| Goal or budget changed | Project, liquidity, seasonality |
| Weekly or monthly cadence | Liquidity, seasonality, regime, anomaly |
| Strategy selected | Regime, liquidity, goal/project evidence |
| User correction | Project, goal, and possibly TFM feedback pipeline |
| Recommendation request | All relevant adapters, plus policy and strategy adapters |

Not every event should invoke the entire adapter set. That is the purpose of routing.

---

## 5. Invocation policy

The inference engine should use a routing policy:

```python
def should_run(adapter, context):
    if not context.permissioned(adapter):
        return False
    if adapter.minimum_history_months > context.history_months:
        return False
    if context.has_active_typed_result(adapter, max_age_days=7):
        return False
    if adapter == "seasonality" and context.history_months < 12:
        return False
    if adapter == "liquidity_pressure" and not context.has_balance_or_income:
        return False
    return True
```

Practical cadence:
- anomaly enrichment: near real time
- project clustering: hourly or after relevant transaction clusters
- liquidity pressure: after income/balance changes or daily
- decision regime: daily or weekly
- seasonality: monthly or after a complete cycle
- strategy execution: only on decision request or material context change

---

## 6. Adapter dependencies

Some adapters depend on other results.

```text
project_clustering
  depends on: transaction semantics, goal context
  feeds: goal association, strategy planning

liquidity_pressure
  depends on: income, obligations, balances
  feeds: strategy admissibility, constraint checks

seasonality
  depends on: recurring history
  feeds: project timing and cash-flow forecast

anomaly
  depends on: household baseline
  feeds: liquidity and project interpretation

decision_regime
  depends on: liquidity, anomaly, budget variance
  feeds: strategy ranking
```

Therefore the engine may invoke a project adapter first to classify events, and only later allow seasonality or liquidity to interpret the result.

No adapter should depend on a recommendation being generated already.

---

## 7. Contradiction handling

The engine must not silently choose the most confident result.

Example contradictions:

- strong project evidence says home maintenance
- cash buffer is below target
- user preference says avoid new debt
- a family travel goal is also active

The fusion engine should emit both interpretations:

```json
{
  "goals": ["home_maintenance", "family_travel"],
  "constraints": ["minimum_cash_buffer"],
  "budget_pressure": "moderate",
  "contradictions": [
    "project urgency conflicts with cash buffer"
  ],
  "strategy_options": [
    "phase_home_maintenance",
    "defer_travel",
    "preserve_liquidity"
  ]
}
```

This becomes the evidence for the user, not a hidden winner.

---

## 8. Orchestration trace

The runtime should record why each adapter ran.

```json
{
  "trace_id": "TRACE-001",
  "tfm_version": "tfm-finance-1.2",
  "context_snapshot_id": "CTX-H22-202611",
  "steps": [
    {
      "step": "encode_sequence",
      "status": "completed",
      "duration_ms": 18
    },
    {
      "step": "route_adapters",
      "selected": ["liquidity", "project", "seasonality", "anomaly", "regime"]
    },
    {
      "step": "fuse_evidence",
      "status": "completed",
      "conflicting_interpretations": 1
    },
    {
      "step": "apply_policy",
      "blocked_strategy_ids": []
    },
    {
      "step": "generate_recommendation",
      "status": "requires_user_confirmation"
    }
  ]
}
```

This is also needed for debugging and evaluation.

---

## 9. TFM involvement in routing

The TFM can inform routing, but it should not select or execute the final policy.

Example route:
- TFM sees that the sequence belongs to a project-like cluster
- Engine invokes project clustering
- TFM also sees balance pressure
- Engine invokes liquidity
- Engine merges both outputs
- Policy decides that high-risk investing is blocked

This keeps adapter discovery model-like while preserving deterministic governance.

---

## 10. Critical recommendation

Use this architecture:

- TFM: sequence understanding and latent context
- Router: event-based adapter selection
- Task adapters: narrow typed predictions
- Fusion engine: merge interpretations and contradictions
- Policy engine: hard constraints and consent
- Strategy engine: eligible options
- Recommendation engine: ranking and explanation
- Feedback system: confirmations, corrections, and outcomes

Do not let adapters directly write to goals or generate recommendations. All writes should pass through the inference engine and an explicit approval or confirmation state.

This architecture allows adapters to be:
- tested independently
- retrained independently
- invoked at appropriate times
- combined without implicit precedence
- governed by the same policy and trace infrastructure

User: >>`A naïve union would say five unrelated things. Fusion should produce one interpretation`
>>`inference engine should use a routing policy`
>>`7. Contradiction handling`
Generate sample test cases for these

GitHub Copilot: 
## 1. Fusion test cases

These cases verify that independent adapter outputs become coherent semantics rather than a loose union.

### Positive fusion case

**Input adapter outputs**

```json
{
  "sequence_id": "SEQ-H22-202611",
  "adapter_outputs": [
    {
      "adapter": "liquidity_pressure",
      "label": "moderate",
      "confidence": 0.76,
      "evidence_event_ids": ["B2", "B3", "D1"]
    },
    {
      "adapter": "seasonality",
      "label": "spring_home_maintenance",
      "confidence": 0.81,
      "evidence_event_ids": ["T1", "T2", "T3"]
    },
    {
      "adapter": "project_clustering",
      "label": "home_maintenance",
      "confidence": 0.91,
      "project_id": "P-91",
      "evidence_event_ids": ["T1", "T2", "T3", "T4"]
    },
    {
      "adapter": "anomaly",
      "label": "elevated_project_spending",
      "confidence": 0.88,
      "evidence_event_ids": ["T3", "T4"]
    },
    {
      "adapter": "decision_regime",
      "label": "cash_conservation",
      "confidence": 0.84,
      "evidence_event_ids": ["R1", "R2", "R3"]
    }
  ]
}
```

**Expected fused result**

```json
{
  "decision_context": {
    "active_project": {
      "id": "P-91",
      "label": "home_maintenance",
      "confidence": 0.91
    },
    "seasonal_pattern": {
      "label": "spring_home_maintenance",
      "confidence": 0.81
    },
    "liquidity_state": {
      "label": "moderate",
      "confidence": 0.76
    },
    "decision_regime": {
      "label": "cash_conservation",
      "confidence": 0.84
    }
  },
  "evidence_summary": [
    "The project cluster supports home maintenance.",
    "The seasonal pattern provides timing context.",
    "Low balances and reduced savings transfers support moderate liquidity pressure.",
    "The elevated spending period supports the cash-conservation regime."
  ],
  "requires_user_confirmation": true
}
```

**Assertions**

- Exactly one active project interpretation wins.
- Seasonality is a modifier, not a second project.
- Liquidity and regime remain separate dimensions.
- No adapter result becomes an independent top-level recommendation.
- One source event may support multiple typed semantics.

---

### Negative fusion case: incompatible outputs

**Input**

```json
{
  "adapter_outputs": [
    {
      "adapter": "project_clustering",
      "label": "home_maintenance",
      "confidence": 0.91
    },
    {
      "adapter": "project_clustering",
      "label": "home_office_setup",
      "confidence": 0.88
    },
    {
      "adapter": "liquidity_pressure",
      "label": "low",
      "confidence": 0.79
    }
  ]
}
```

**Expected**

```json
{
  "fusion_status": "ambiguous",
  "candidate_semantics": [
    {"label": "home_maintenance", "confidence": 0.91},
    {"label": "home_office_setup", "confidence": 0.88}
  ],
  "liquidity_state": {"label": "low", "confidence": 0.79},
  "dominant_interpretation": null,
  "requires_user_confirmation": true,
  "reason": "Two project candidates have closely comparable confidence."
}
```

**Assertions**

- The engine must not select the top result merely by maximum confidence.
- It must return candidates and preserve ambiguity.
- The liquidity result must remain available for constraint evaluation.

---

### Negative fusion case: contradictory states

**Input**

```json
{
  "adapter_outputs": [
    {"adapter": "liquidity_pressure", "label": "low", "confidence": 0.91},
    {"adapter": "decision_regime", "label": "aggressive_growth", "confidence": 0.87},
    {"adapter": "anomaly", "label": "no_recent_deterioration", "confidence": 0.72}
}
```

**Expected**

```json
{
  "fusion_status": "contradictory",
  "liquidity_state": "low",
  "decision_regime": "aggressive_growth",
  "contradictions": [
    "aggressive_growth preference conflicts with low-liquidity evidence"
  ],
  "recommendation_allowed": false,
  "requires_user_confirmation": true
}
```

**Assertions**

- Contradiction is retained rather than overwritten.
- No strategy such as aggressive investment is automatically ranked.
- Policy can block or defer the recommendation.

---

## 2. Routing-policy test cases

### No liquidity history

```json
{
  "test_id": "ROUTE-001",
  "context": {
    "history_months": 2,
    "has_income": false,
    "has_balance": false,
    "active_goals": ["home_maintenance"]
  },
  "expected_adapters": [
    "project_clustering",
    "anomaly"
  ],
  "blocked_adapters": [
    "liquidity_pressure",
    "seasonality",
    "decision_regime"
  ]
}
```

**Assertions**

- A short history prevents seasonality.
- Missing income and balance prevent liquidity pressure.
- The project adapter may still run if transaction history is sufficient.

---

### Sufficient transaction but short household history

```json
{
  "test_id": "ROUTE-002",
  "context": {
    "history_months": 4,
    "new_transaction_types": ["paint", "hardware", "contractor"],
    "active_project": "P-91",
    "permissioned_adapters": ["project_clustering", "anomaly"]
  },
  "expected_adapters": ["project_clustering", "anomaly"],
  "blocked_adapters": ["seasonality"]
}
```

**Assertions**

- Project detection can run without twelve months of history.
- Seasonality is blocked by insufficient annual evidence.

---

### Cached result prevents duplicate execution

```json
{
  "test_id": "ROUTE-003",
  "context": {
    "active_typed_results": {
      "liquidity_pressure": {
        "age_days": 1,
        "state": "moderate"
      }
    },
    "material_change": false
  },
  "expected_adapters": [],
  "reuse_typed_results": ["liquidity_pressure"]
}
```

**Assertions**

- No fresh liquidity adapter call occurs for a minor event.
- The engine reuses the cached typed result.
- The result is marked as cached and still carries its provenance.

---

### Material liquidity event overrides cache

```json
{
  "test_id": "ROUTE-004",
  "context": {
    "active_typed_results": {
      "liquidity_pressure": {"age_days": 1, "state": "moderate"}
    },
    "material_change": {
      "type": "income_drop",
      "percent": 35
    }
  },
  "expected_adapters": ["liquidity_pressure", "decision_regime"],
  "invalidate_cached": ["liquidity_pressure"]
}
```

**Assertions**

- The routing policy treats an income drop as a material context change.
- It invalidates stale liquidity state.
- The decision-regime adapter receives the updated state.

---

### Permission-denied routing

```json
{
  "test_id": "ROUTE-005",
  "context": {
    "household_id": "H-22",
    "sensitive_adapter_permissions": {
      "seasonality": false,
      "community_benchmark": false
    }
  },
  "expected_adapters": ["anomaly", "project_clustering"],
  "blocked_adapters": ["seasonality", "community_benchmark"]
}
```

**Assertions**

- Blocked adapters do not receive raw account or household data.
- Routing emits an audit entry for the blocked adapter.
- A result from a blocked adapter cannot appear in downstream output.

---

## 3. Contradiction-handling test cases

### Goal conflicts with liquidity constraint

**Input**

```json
{
  "household_id": "H-22",
  "active_goal": {
    "goal_id": "G-02",
    "name": "home_maintenance",
    "priority": 2
  },
  "constraint": {
    "type": "minimum_cash_buffer",
    "required": 5000
  },
  "state": {
    "current_cash_buffer": 4200
  },
  "project_candidate": {
    "name": "home_maintenance",
    "unpaid_amount": 2800
  }
}
```

**Expected**

```json
{
  "contradiction_id": "C-001",
  "type": "goal_vs_constraint",
  "status": "unresolved",
  "resolution_candidates": [
    "phase_project_across_two_months",
    "reduce_project_scope",
    "preserve_liquidity_and_delay_project"
  ],
  "recommendation_allowed": false,
  "requires_user_confirmation": true
}
```

**Assertions**

- The goal is not silently deprioritized.
- The constraint is not silently waived.
- A deferment or phased strategy is evaluated.

---

### Strategy conflicts with user preference

**Input**

```json
{
  "strategy_candidate": "debt_first",
  "model_reason": "high_credit_apr",
  "user_preference": {
    "cash_buffer_priority": "keep_3000_available",
    "risk_tolerance": "conservative"
  },
  "calculated_state": {
    "projected_cash_buffer": 2200
  }
}
```

**Expected**

```json
{
  "contradiction_id": "C-002",
  "type": "strategy_vs_user_preference",
  "status": "unresolved",
  "resolution_candidates": [
    "debt_first_with_cash_buffer_guardrail",
    "hybrid_debt_and_buffer",
    "preserve_buffer_then_reassess"
  ],
  "must_not_emit": "pay_down_debt_at_all_costs"
}
```

**Assertions**

- The recommended amount is constrained by the user’s cash-buffer preference.
- A conservative alternative is considered.
- The original preference remains visible in the evidence trail.

---

### Seasonal evidence conflicts with current anomaly

**Input**

```json
{
  "seasonality_result": {
    "label": "winter_travel",
    "confidence": 0.84
  },
  "anomaly_result": {
    "label": "unexpected_travel_spending",
    "confidence": 0.78
  },
  "history": [
    {"month": "2025-12", "category": "travel", "amount": 2100},
    {"month": "2026-12", "category": "travel", "amount": 2300}
  ]
}
```

**Expected**

```json
{
  "contradiction_status": "resolved",
  "seasonal_pattern_confirmed": true,
  "anomaly_classification": "not_an_anomaly",
  "reason": "Travel spending recurs at the same annual time and is consistent with prior behavior."
}
```

**Assertions**

- A seasonal explanation supersedes an otherwise valid magnitude anomaly.
- The engine records why the anomaly result was rejected.
- The final output does not report both “seasonal” and “unexpected” as independent facts.

---

### Ambiguous evidence: no contradiction but insufficient certainty

```json
{
  "test_id": "CONTRA-004",
  "adapter_outputs": [
    {"adapter": "project_clustering", "label": "home_maintenance", "confidence": 0.68},
    {"adapter": "seasonality", "label": "unrecognized", "confidence": 0.20}
  ]
}
```

**Expected**

```json
{
  "contradiction_status": "insufficient_evidence",
  "recommendation_allowed": false,
  "requires_user_confirmation": true,
  "message": "The recent purchases may belong to a project, but the pattern cannot be confirmed."
}
```

---

## Shared fusion hard gates

Every fusion test should verify:

- Valid typed schema
- Every evidence ID exists
- Confidence is within $[0,1]$
- Same evidence cannot be silently dropped
- Contradictions are represented explicitly
- Ambiguous candidates are not collapsed into certainty
- Recommendations cannot bypass unresolved hard constraints
- A user confirmation state is present when certainty is insufficient
- The orchestration trace identifies routing and fusion decisions

A strong test suite should include golden cases, ambiguous cases,
