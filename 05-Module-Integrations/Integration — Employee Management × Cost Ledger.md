# Integration — Employee Management × Cost Ledger

#integration #module/employee-mgmt #module/cost-ledger #status/active

> How employee lifecycle events translate into cost rules.

## Data Flow

The [[Employee Management — Overview]] module triggers cost rule creation/closing. The [[Cost Ledger — Overview]] tracks the financial impact.

### Employee Hired → Create Cost Rule

```
Event: emp-002 hired on 2025-02-10, salary 45,000/month
Action: Create CostRule(
  entity_type=employee, entity_id=emp-002,
  cost_type=salary, amount=45000, frequency=monthly,
  proration_strategy=daily_prorate,
  effective_from=2025-02-10
)
Result: Feb cost auto-prorated to 19/28 days = 30,535.71
```

### Salary Change → Close + Create

See [[ADR-001 — Close and Replace, Never Mutate]].

```
Event: emp-002 raise to 50,000 effective April 1
Actions:
  1. Close old rule: effective_until=2025-03-31
  2. Create new rule: amount=50000, effective_from=2025-04-01
```

### Employee Resigned/Terminated → Close Rule

```
Event: emp-002 leaves on 2025-05-20
Action: Close rule: effective_until=2025-05-20
Result: May cost auto-prorated to 20/31 days
```

### Employee Rehired → New Rule (don't reopen)

```
Event: emp-002 rehired on 2025-09-01 at 55,000
Action: Create new CostRule(effective_from=2025-09-01, amount=55000)
Result: Gap from May 20 to Sep 1 correctly shows zero cost
```

### Contractor (Daily Rate) → Daily Frequency Rule

```
Event: emp-003 hired as daily contractor at 2,000/day
Action: Create CostRule(frequency=daily, proration_strategy=none, amount=2000)
Result: Cost = 2000 × number of active days in query range
```

## Additional Cost Types

Beyond salary, employees may have:

- Social security / tax contributions → separate monthly rule
- Benefits (health insurance, etc.) → separate monthly rule
- One-time signing bonus → [[Cost Event]], not a rule

Each is a separate [[Cost Rule]] so they can change independently.

## Related

- [[Employee Management — Overview]]
- [[Cost Ledger — Overview]]
- [[Integration — Cost Ledger × Delivery Costing]] — driver cost in deliveries
- [[ADR-003 — Month-by-Month Proration]] — why Feb and Dec salary costs differ