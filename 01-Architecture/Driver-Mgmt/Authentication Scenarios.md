# Authentication Scenarios

#module/driver-mgmt #module/auth #status/active

> End-to-end flows for creating accounts and logging in across web and mobile apps.

## Identities

| Entity | Purpose | Required for |
|---|---|---|
| [[Driver]] | Operational record (license, rates) | Being assigned to Work Orders |
| User | Auth record (username, password, role) | Logging in to web or mobile |
| Role | Permission set | Controller, planner, accountant, viewer, `driver-mobile`, etc. |

`User.driver_id` is nullable. When set, the User belongs to a Driver and may use the mobile app. When NULL, the User is office staff (controller / planner / etc.) — see [[ADR-013 — Driver as Cost Entity, Not Employee]].

## Login Endpoint

A single endpoint serves both apps:

```
POST /auth/login { username, password }
  → verify User row exists, password matches, status=active
  → issue JWT { user_id, username, role, driver_id }
```

Each app gates by JWT contents:

- **Web app** — accepts any role except `driver-mobile`-only users. Routes the user by role.
- **Mobile app** — rejects login if `driver_id IS NULL`. All mobile API calls are scoped to `driver_id` from the JWT.

## Scenarios

### A. Create controller (web only)

```
1. Admin opens "Create User" form
2. Fields: username, password, role (controller / planner / accountant / ...)
3. driver_id = NULL
4. INSERT User
```

No Driver row. Cannot log into mobile app.

### B. Create driver, no app access

```
1. Admin opens "Create Driver" form
2. Fields: employee_code, name, phone, license_*, monthly salary, rates
3. INSERT Driver
4. (no User row)
```

Driver visible in WO assignment dropdown. Cannot log in anywhere.

### C. Create driver, with app access

```
1. Admin opens "Create Driver" form
2. Fill driver fields + check "Create mobile login"
3. Enter initial password
4. Transaction:
   a. INSERT Driver
   b. INSERT User(
         username      = driver.employee_code,
         password_hash = ...,
         role          = driver-mobile,
         driver_id     = driver.driver_id
      )
```

Driver can log into the mobile app immediately. `username` is locked to match `employee_code`.

### D. Existing driver gets app access later

```
1. Admin opens Driver detail
2. Click "Enable mobile login"
3. Enter initial password
4. INSERT User(
      username      = driver.employee_code,
      password_hash = ...,
      role          = driver-mobile,
      driver_id     = this driver
   )
```

### E. Revoke driver app access

```
1. Admin opens Driver detail
2. Click "Disable mobile login"
3. UPDATE User SET status=disabled WHERE driver_id = this   (soft)
```

Soft-disable is the default — preserves audit trail, keeps Work Order references intact. Hard delete is reserved for clear data-entry mistakes.

### F. Driver also becomes controller

The system uses **separate User rows** for separate roles (decision F1):

```
Existing driver DRV-001 with mobile User username=DRV-001
Person becomes a controller as well:

  INSERT User(
     username   = "drv001-ctrl",   -- different username
     password   = ...,
     role       = controller,
     driver_id  = NULL
  )
```

Two distinct logins, two distinct passwords, two distinct audit trails. The driver continues to log into the mobile app with the original `DRV-001` credentials; controller work happens on the web app with the new credentials.

This separation also means a controller-only User cannot accidentally pick up driver-mobile permissions via role escalation — each app login is scoped to one identity.

### G. Controller takes on driving

```
1. Admin opens "Create Driver" for that person
2. INSERT Driver (license, rates, etc.)
3. Create a NEW mobile User per scenario D — do not modify the existing controller User
```

Same separation as F.

### H. Web login flow

```
POST /auth/login { username, password }
  → verify User; password match; status=active
  → JWT { user_id, role, driver_id }
  → web client routes by role
```

A web client signed in with a `driver-mobile`-only User is shown "no permitted module" — the mobile role does not grant any web permissions.

### I. Mobile login flow

```
POST /auth/login { username, password }    -- same endpoint as web
  → verify User
  → JWT { user_id, role, driver_id }

Mobile client checks JWT.driver_id:
  - NULL  → reject "no driver profile"
  - set   → proceed
  
All mobile API calls scope by jwt.driver_id:
  - GET /mobile/work_orders         → WHERE WorkOrderDriver.driver_id = jwt.driver_id
  - GET /mobile/compensation        → for jwt.driver_id only
  - POST /mobile/fuel_log           → driver_id = jwt.driver_id (cannot spoof)
```

## Edge Cases

| Case | Handling |
|---|---|
| Driver disabled mid-trip | `User.status=disabled`. Existing JWT valid until expiry; gateway re-checks status on every request → 401 once disabled. WO continues; controller manages remaining stops or migrates per [[ADR-010 — Work Order Migration on Failure]]. |
| Driver record deleted | Block hard delete. Soft-disable only — Work Order history references the row. |
| Employee code change after creation | Disallowed. The code is the username and the public-facing identifier; treat as immutable after first use. |
| Same phone, two drivers | Allowed. Phone is not unique. `employee_code` is the unique key. |
| Forgot password | Controller resets via admin UI in v1. Driver self-reset (SMS OTP) deferred. |
| Same person, controller + driver | Two User rows per scenario F. |

## JWT Contents Summary

```json
{
  "user_id":   "USR-007",
  "username":  "DRV-001",
  "role":      "driver-mobile",
  "driver_id": "DRV-001",
  "exp":       1735689600
}
```

Every protected API derives the acting Driver from `jwt.driver_id` directly. The mobile client never sends `driver_id` in request bodies — the server reads it from the JWT to prevent spoofing.

## Identity Matrix

| Persona | Driver row | User row(s) | role | driver_id |
|---|---|---|---|---|
| Controller / planner / CEO | – | 1 | controller / planner / ... | NULL |
| Driver, no mobile login | 1 | – | – | – |
| Driver with mobile login | 1 | 1 | driver-mobile | set |
| Driver who is also a controller (F1) | 1 | 2 | one mobile, one web | one set, one NULL |

## Related

- [[Driver]]
- [[Driver Management — Overview]]
- [[ADR-013 — Driver as Cost Entity, Not Employee]]
- [[ADR-010 — Work Order Migration on Failure]]
