# Open Feedback API — Test Data

Base URL: `{{base_url}}/admin/api/v1/feedback/open`
No auth required on either endpoint.

---

## API 1 — Check Pending Feedback
`GET /pending?mobile_number=<mobile>`

### PENDING (hasPendingFeedback: true)

| # | Mobile | Name | Role | Segment | Level | surveyId | l1Drivers |
|---|--------|------|------|---------|-------|----------|-----------|
| 1 | `2929828282` | Test | HO_RO_COMMERCIAL | KEY | GROUP | 20260627G | test |
| 2 | `8826266047` | Balveer Singh | HO_RO_COMMERCIAL | KEY | GROUP | 20260627G | test |
| 3 | `9775937767` | Anurag Aggarwal | HO_RO_COMMERCIAL | KEY | GROUP | 20260627G | test |
| 4 | `9214860857` | Kapil Jain | HO_RO_COMMERCIAL | KEY | GROUP | 20260627G | test |
| 5 | `8976985677` | Hiren Ji | PLANNING_COORDINATOR | KEY | SITE | 20260627 | Customer Relationship Management, Delivery Efficiency, Ordering Process and Availability |
| 6 | `7350000000` | Kirandeep Singh | PLANNING_COORDINATOR | KEY | SITE | 20260627 | same as above |
| 7 | `9844646078` | Ramesh Maloo | PLANNING_COORDINATOR | KEY | SITE | 20260627 | same as above |
| 8 | `6367823542` | GOVIND GUPTA | PLANNING_COORDINATOR | KEY | SITE | 20260627 | same as above |

Expected response:
```json
{
  "hasPendingFeedback": true,
  "surveyId": "20260627G",
  "customerName": "...",
  "siteId": "...",
  "groupCode": "...",
  "lob": "...",
  "businessSegment": "KEY",
  "touchpointRole": "HO_RO_COMMERCIAL",
  "l1Drivers": ["test"]
}
```

---

### ALREADY_SUBMITTED (hasPendingFeedback: false)

| # | Mobile | Name | Role | Survey already filled |
|---|--------|------|------|-----------------------|
| 9 | `9978385849` | Jitendra Panchal | HO_RO_PROCUREMENT | 20260627G (NPS=2) |
| 10 | `9376710358` | Birender Singh | HO_RO_PROCUREMENT | 20260627G (NPS=2) |

Expected response:
```json
{ "hasPendingFeedback": false, "reason": "ALREADY_SUBMITTED" }
```

---

### NO_SURVEY_TRIGGERED (hasPendingFeedback: false)

| # | Mobile | Name | Role | Reason |
|---|--------|------|------|--------|
| 11 | `2228357000` | P ANAND | SITE_STORES_MANAGER | Active touchpoint, zero outbox entries |
| 12 | `2240647500` | ARUN KUMAR AGARWAL | SITE_STORES_MANAGER | Same |

Expected response:
```json
{ "hasPendingFeedback": false, "reason": "NO_SURVEY_TRIGGERED" }
```

---

### NO_TOUCHPOINT (hasPendingFeedback: false)

| # | Mobile | Reason |
|---|--------|--------|
| 13 | `9999999999` | Not in MH_TOUCHPOINT_MAPPING |
| 14 | `1234567890` | Not in MH_TOUCHPOINT_MAPPING |

Expected response:
```json
{ "hasPendingFeedback": false, "reason": "NO_TOUCHPOINT" }
```

---

## API 2 — Submit Open Feedback
`POST /submit`

```json
{
  "mobileNumber": "<mobile>",
  "feedbackId": "<any string, not stored>",
  "npsScore": <0-10>,
  "l1TemplateId": "<one of the l1Drivers from check-pending response>"
}
```

### Success Cases

| # | Mobile | npsScore | l1TemplateId | NPS Category | Saved in table |
|---|--------|----------|--------------|--------------|----------------|
| 15 | `8968083334` | 9 | test | PROMOTER | MH_CUSTOMER_FEEDBACK_KEY |
| 16 | `8697543258` | 7 | test | PASSIVE | MH_CUSTOMER_FEEDBACK_KEY |
| 17 | `8802605727` | 4 | Delivery Efficiency | DETRACTOR | MH_CUSTOMER_FEEDBACK_KEY |
| 18 | `9750633551` | 10 | Customer Relationship Management | PROMOTER | MH_CUSTOMER_FEEDBACK_KEY |
| 19 | `7742191465` | 8 | Ordering Process and Availability | PASSIVE | MH_CUSTOMER_FEEDBACK_KEY |

NPS category rules:
- Score 9–10 → PROMOTER
- Score 7–8 → PASSIVE
- Score 0–6 → DETRACTOR

Expected response:
```json
{ "feedbackId": 1234, "customerCode": "5678" }
```

Verify in DB:
```sql
SELECT * FROM MH_CUSTOMER_FEEDBACK_KEY WHERE FEEDBACK_CHANNEL = 'OPEN_API' ORDER BY ID DESC LIMIT 5;
```

---

### Error Cases

| # | Mobile | npsScore | l1TemplateId | Expected | Error |
|---|--------|----------|--------------|----------|-------|
| 20 | `8968083334` | 6 | test | 409 | FEEDBACK_ALREADY_SUBMITTED (re-submit after #15) |
| 21 | `9999999999` | 8 | test | 400/500 | No active touchpoint found |
| 22 | `2228357000` | 8 | Availability | 500 | No survey found for mobile |
| 23 | any | 11 | test | 400 | npsScore must be ≤ 10 |
| 24 | *(omit)* | 8 | test | 400 | mobileNumber: must not be blank |
| 25 | `9413376905` | 7 | *(omit)* | 400 | l1TemplateId: must not be blank |
| 26 | `9413376905` | -1 | Delivery Efficiency | 400 | npsScore must be ≥ 0 |

---

## L1 Driver Reference

| Role | Segment | Valid l1TemplateId values |
|------|---------|--------------------------|
| HO_RO_COMMERCIAL | KEY | `test` |
| HO_RO_PROCUREMENT | KEY | `Test 2` |
| PLANNING_COORDINATOR | KEY | `Customer Relationship Management`, `Delivery Efficiency`, `Ordering Process and Availability` |
| SITE_STORES_MANAGER | KEY | `Availability`, `Customer Relationship Management`, `CUSTOMER SERVICE`, `Delivery Efficiency`, `Ordering Process` |

---

## Test Execution Order

1. Run #1–14 (check-pending) in any order — read-only, safe to repeat
2. Run #15–19 (submit) once each — each writes a new row
3. Run #20 immediately after #15 — same mobile, tests dedup
4. Run #21–26 — error cases, safe to repeat
