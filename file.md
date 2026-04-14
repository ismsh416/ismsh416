# 🏢 EntityType Controller — Full Business Explanation

---

## 🌍 What is this system about?

Think of it like a **company address book** used by a big organization like **CSC Global**.

CSC Global manages thousands of companies around the world —
Google, Microsoft, Apple, Amazon, etc.

Different **departments inside CSC** (HR, Finance, Legal, Entity Management)
all use **different internal codes** to refer to the same company type.

> 🎯 The **Entity Integration Service** is the **translator** that says:
> **"Hey, Google's Finance code and Google's HR code are both referring to the same company type."**

---

## 🧠 Two Key Concepts (Very Simple)

### 1️⃣ Global Entity Type (Canonical Type) — The MASTER label
> This is the **one true, universal name** for a company type.
> Like a **"dictionary definition"** that everyone agrees on.

| Example | Canonical Name |
|---------|---------------|
| Limited Liability Company | `LLC` |
| Private Limited Company | `PVT` |
| Public Limited Company | `PLC` |

---

### 2️⃣ Domain Entity Type — The DEPARTMENT-SPECIFIC label
> This is how **each internal system** refers to that same company type using **their own code**.

| Department | Their Code | Meaning |
|-----------|-----------|---------|
| Finance System (EM) | `101` | LLC |
| HR System (CSC) | `XY-77` | LLC |
| Legal System | `LTD-A` | LLC |

> So **one Global Type** (`LLC`) can have **many Domain Types** (one per department/system).

---

## 🔌 The Two API Endpoints — POST and PATCH

---

## ✅ POST `/entityTypes` — Register a New Company Type Mapping

### Real-World Story:
> CSC is onboarding **Google LLC** into their system.
>
> The Finance department uses code `"101"` for LLC type in their EM system.
>
> A CSC Admin calls POST to say:
> **"In the Finance system (EM), code 101 = LLC. Please register this mapping."**

### What happens step by step:

```
Admin sends:
{
  "globalTypeCode": "LLC",          ← The universal name (master label)
  "domainTypeCode": "101",          ← Finance's internal code
  "identifierValue": "101",         ← The unique ID Finance uses
  "systemCode": "EM",               ← Which system this belongs to
  "description": "Limited Liability Company"
}
```

**Step 1** → Check: Does a mapping for `identifierValue=101` in `systemCode=EM` already exist?
- YES → ❌ **Reject with 409 Conflict** — "This mapping already exists!"
- NO  → ✅ Continue

**Step 2** → Find or create the Global Type (`LLC`):
- Already exists? → Reuse it ♻️
- Doesn't exist? → Create a new Global Type called `LLC` 🆕

**Step 3** → Create the new Domain mapping (Finance's code 101 = LLC)

**Step 4** → Return the full picture:
```json
{
  "id": "5",
  "canonicalName": "LLC",
  "createdDateTime": "2026-01-01T00:00:00Z",
  "domainEntityTypes": [
    {
      "id": 10,
      "domainTypeCode": "101",
      "externalTypeId": "101",
      "description": "Limited Liability Company",
      "domainCode": "EM",
      "fetchedDateTime": "..."
    }
  ]
}
```

---

### ❌ What if you POST the same thing twice?

**Before our fix:** System silently overwrote the old record. 😱

**After our fix:**
```json
409 Conflict
{
  "error": "DomainEntityType already exists with identifierValue: 101 and systemCode: EM"
}
```
> Like trying to **register the same student roll number twice** in school —
> the system now says **"This roll number is already taken!"** instead of quietly replacing the old student.

---

## ✅ PATCH `/entityTypes/{id}` — Update an Existing Mapping

### Real-World Story:
> Google's Finance department changes their internal LLC code from `"101"` to `"LLC-GOLD"`.
>
> A CSC Admin needs to update that mapping in the system.
>
> They call PATCH with only the fields that changed.

### What makes PATCH special:
> 📌 PATCH means **"only change what I send you — leave everything else alone"**
>
> Unlike a full update, if you don't send a field → it stays unchanged.
> If you send `null` → it is also ignored (the old value is kept).

```
Admin sends PATCH /entityTypes/10:
{
  "identifierValue": "LLC-GOLD",      ← Update this
  "description": "LLC Gold Tier"      ← Update this
  (everything else = not sent = unchanged)
}
```

**Step 1** → Find record #10. If not found → ❌ **404 Not Found**

**Step 2** → Check: Would the new `identifierValue + systemCode` clash with another existing record?
- YES, belongs to a DIFFERENT record → ❌ **409 Conflict** — "That code is already used by Apple!"
- YES, same record → ✅ Fine (you're just updating it to the same values)
- NO clash → ✅ Continue

**Step 3** → Apply only the non-null fields:
```
identifierValue: "LLC-GOLD"   ← UPDATED ✅
description: "LLC Gold Tier"  ← UPDATED ✅
domainTypeCode: (not sent)    ← UNCHANGED 🔒
systemCode: (not sent)        ← UNCHANGED 🔒
```

**Step 4** → Return the full updated picture (now includes `domainTypeCode` in response ✅)

---

### ❌ What if you PATCH with duplicate values?

**Scenario:**
- Record #10 = Google → EM system → code `101`
- Record #15 = Apple  → EM system → code `202`

Admin accidentally sends PATCH on Record #10 with `identifierValue: "202"` (Apple's code):
```json
409 Conflict
{
  "error": "DomainEntityType already exists with identifierValue: 202 and systemCode: EM"
}
```
> Like trying to give **Google's employee** the same **ID badge number** that Apple's employee already has.
> The system now correctly **blocks it**.

---

### ❌ What if you PATCH with all null values?

```json
{
  "globalTypeCode": null,
  "domainTypeCode": null,
  "identifierValue": null,
  "systemCode": null,
  "description": null
}
```

**Result:** Nothing changes. Record stays exactly as-is. Returns 200 OK with current values.
> Like telling a bank:
> **"Update my account details"** but providing **no new information**.
> The bank just says **"OK, nothing to change"** and shows you your existing details.

---

## 🔐 Who Can Use These APIs?

| Role | What They Can Do |
|------|-----------------|
| `ead_admin` | POST & PATCH ✅ |
| `ET_API_CLIENT` | POST & PATCH ✅ |
| No valid token | ❌ 401 Unauthorized |

> Before the fix, only `ead_admin` could call these APIs.
> Regular API users (`ET_API_CLIENT`) were getting **403 Forbidden** even for valid requests.
> **Fixed: Both roles now have access.**

---

## 🗺️ Wrong URL Behaviour

| Scenario | Before | After |
|----------|--------|-------|
| `/entityTypess` (typo) | 403 Forbidden ❌ | 404 Not Found ✅ |
| `/entityTypes/99` (ID not in DB) | 200 OK (crash) | 404 Not Found ✅ |

> Hitting a wrong door should say **"This room doesn't exist"** — not **"You're not allowed in"**.

---

## 🗂️ Full Flow — Big Picture

```
                        ┌─────────────────────────────────┐
                        │        CSC Global System         │
                        └──────────────┬──────────────────┘
                                       │
              ┌────────────────────────▼────────────────────────┐
              │           Entity Integration Service             │
              │                                                  │
              │  POST /entityTypes  ──► Register new mapping     │
              │  PATCH /entityTypes/{id} ──► Update mapping      │
              └────────────────┬─────────────────┬──────────────┘
                               │                 │
               ┌───────────────▼──┐     ┌────────▼──────────────┐
               │  Canonical Type  │     │   Domain Entity Type   │
               │  (Master label)  │     │ (Per-department code)  │
               │                  │     │                         │
               │  id=5, name=LLC  │◄────│ id=10, code=101, EM    │
               │                  │     │ id=11, code=XY77, CSC  │
               └──────────────────┘     └─────────────────────────┘
```

---

## 📋 Response HTTP Status Codes — Quick Reference

| Situation | HTTP Code | Meaning |
|-----------|-----------|---------|
| Created successfully | `201 Created` | ✅ All good, new record saved |
| Updated successfully | `200 OK` | ✅ All good, record updated |
| Duplicate detected | `409 Conflict` | ❌ This already exists, stop |
| Record not found | `404 Not Found` | ❌ No record with that ID |
| Wrong URL | `404 Not Found` | ❌ That endpoint doesn't exist |
| Validation failed | `400 Bad Request` | ❌ You forgot a required field |
| Not logged in | `401 Unauthorized` | ❌ Who are you? Login first |
| Wrong role | `403 Forbidden` | ❌ You don't have permission |
