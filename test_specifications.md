# CFO BOT — AI Test Specifications

## Scope

These test specifications cover the `billing_engine` module — the core engine for calculating computing resource costs.

**Base formula:**
```
Cost ($) = Compute Hours × Rate ($/hour)
```

---

## Unit Tests — Billing Calculator

### TC-001: Standard cost calculation

| Parameter | Value |
|---|---|
| **Test ID** | TC-001 |
| **Name** | Standard compute cost calculation |
| **Priority** | 🔴 Critical |

**Input:**
```json
{ "compute_hours": 100, "rate_per_hour": 0.05 }
```
**Expected Result:**
```
100 × 0.05 = $5.00
```
**Assertion:** `result.total_cost == 5.00`

---

### TC-002: Zero hours

| Parameter | Value |
|---|---|
| **Test ID** | TC-002 |
| **Name** | Zero compute hours → zero cost |
| **Priority** | 🟡 High |

**Input:**
```json
{ "compute_hours": 0, "rate_per_hour": 0.05 }
```
**Expected Result:**
```
0 × 0.05 = $0.00
```
**Assertion:** `result.total_cost == 0.00`

---

### TC-003: Fractional hours

| Parameter | Value |
|---|---|
| **Test ID** | TC-003 |
| **Name** | Fractional hours calculation |
| **Priority** | 🟡 High |

**Input:**
```json
{ "compute_hours": 1.5, "rate_per_hour": 0.05 }
```
**Expected Result:**
```
1.5 × 0.05 = $0.075
```
**Assertion:** `abs(result.total_cost - 0.075) < 0.0001`

---

### TC-004: Large-scale compute

| Parameter | Value |
|---|---|
| **Test ID** | TC-004 |
| **Name** | Large-scale compute (1,000,000 hours) |
| **Priority** | 🟡 High |

**Input:**
```json
{ "compute_hours": 1000000, "rate_per_hour": 0.05 }
```
**Expected Result:**
```
1,000,000 × 0.05 = $50,000.00
```
**Assertion:** `result.total_cost == 50000.00`

---

### TC-005: Custom rate

| Parameter | Value |
|---|---|
| **Test ID** | TC-005 |
| **Name** | Custom rate (GPU compute) |
| **Priority** | 🟠 Medium |

**Input:**
```json
{ "compute_hours": 10, "rate_per_hour": 2.50 }
```
**Expected Result:**
```
10 × 2.50 = $25.00
```
**Assertion:** `result.total_cost == 25.00`

---

### TC-006: Negative hours (edge case)

| Parameter | Value |
|---|---|
| **Test ID** | TC-006 |
| **Name** | Negative hours → ValidationError |
| **Priority** | 🟠 Medium |

**Input:**
```json
{ "compute_hours": -10, "rate_per_hour": 0.05 }
```
**Expected Result:**
```
raises ValidationError: "compute_hours must be >= 0"
```
**Assertion:** `pytest.raises(ValidationError)`

---

### TC-007: Batch calculation for multiple resources

| Parameter | Value |
|---|---|
| **Test ID** | TC-007 |
| **Name** | Batch billing — multiple resources |
| **Priority** | 🟠 Medium |

**Input:**
```json
[
  { "resource": "CPU",     "compute_hours": 100, "rate_per_hour": 0.05 },
  { "resource": "GPU",     "compute_hours": 10,  "rate_per_hour": 2.50 },
  { "resource": "Storage", "compute_hours": 500, "rate_per_hour": 0.01 }
]
```
**Expected Result:**
```
CPU:     100  × 0.05 = $5.00
GPU:     10   × 2.50 = $25.00
Storage: 500  × 0.01 = $5.00
─────────────────────────────
TOTAL:               = $35.00
```
**Assertion:** `result.grand_total == 35.00`

---

### TC-008: Currency conversion (USD → KZT)

| Parameter | Value |
|---|---|
| **Test ID** | TC-008 |
| **Name** | Currency conversion USD to KZT |
| **Priority** | 🟢 Low |

**Input:**
```json
{ "compute_hours": 100, "rate_per_hour": 0.05, "currency": "KZT", "exchange_rate": 450 }
```
**Expected Result:**
```
100 × 0.05 = $5.00 → 5.00 × 450 = ₸2,250.00
```
**Assertion:** `result.converted_cost == 2250.00`

---

## Integration Tests — API Endpoints

### TC-101: POST /calculate — successful request

**Input:**
```http
POST /calculate
Content-Type: application/json

{ "compute_hours": 100, "rate_per_hour": 0.05 }
```
**Expected Result:**
```json
{ "status": "ok", "total_cost": 5.00, "currency": "USD" }
```
**Assertion:** `response.status_code == 200` and `response.json()["total_cost"] == 5.00`

---

### TC-102: POST /calculate — invalid request

**Input:**
```http
POST /calculate
Content-Type: application/json

{ "compute_hours": "abc", "rate_per_hour": 0.05 }
```
**Expected Result:**
```json
{ "status": "error", "detail": "compute_hours must be a number" }
```
**Assertion:** `response.status_code == 422`

---

## pytest Implementation Template

```python
# tests/test_billing_engine.py
import pytest
from billing_engine import calculate_cost, BillingResult

class TestBillingEngine:

    def test_tc001_standard_calculation(self):
        """TC-001: 100 hours × $0.05 = $5.00"""
        result = calculate_cost(compute_hours=100, rate_per_hour=0.05)
        assert result.total_cost == 5.00

    def test_tc002_zero_hours(self):
        """TC-002: 0 hours → $0.00"""
        result = calculate_cost(compute_hours=0, rate_per_hour=0.05)
        assert result.total_cost == 0.00

    def test_tc003_fractional_hours(self):
        """TC-003: 1.5 hours × $0.05 = $0.075"""
        result = calculate_cost(compute_hours=1.5, rate_per_hour=0.05)
        assert abs(result.total_cost - 0.075) < 0.0001

    def test_tc004_large_scale(self):
        """TC-004: 1,000,000 hours × $0.05 = $50,000"""
        result = calculate_cost(compute_hours=1_000_000, rate_per_hour=0.05)
        assert result.total_cost == 50_000.00

    def test_tc005_custom_rate(self):
        """TC-005: 10 hours × $2.50 = $25.00"""
        result = calculate_cost(compute_hours=10, rate_per_hour=2.50)
        assert result.total_cost == 25.00

    def test_tc006_negative_hours_raises(self):
        """TC-006: Negative hours → ValidationError"""
        with pytest.raises(ValueError, match="compute_hours must be >= 0"):
            calculate_cost(compute_hours=-10, rate_per_hour=0.05)
```

---

## Test Coverage Summary

| Test ID | Scenario | Priority | Type |
|---|---|---|---|
| TC-001 | Standard 100h × $0.05 = $5 | 🔴 Critical | Unit |
| TC-002 | Zero hours | 🟡 High | Unit |
| TC-003 | Fractional hours | 🟡 High | Unit |
| TC-004 | Large-scale (1M hours) | 🟡 High | Unit |
| TC-005 | Custom rate (GPU) | 🟠 Medium | Unit |
| TC-006 | Negative hours → error | 🟠 Medium | Unit |
| TC-007 | Batch multi-resource | 🟠 Medium | Unit |
| TC-008 | Currency conversion | 🟢 Low | Unit |
| TC-101 | API success response | 🔴 Critical | Integration |
| TC-102 | API validation error | 🟡 High | Integration |
