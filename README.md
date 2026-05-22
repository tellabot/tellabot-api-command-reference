# Tell A Bot — API Command Reference

API reference for the [Tell A Bot](https://www.tellabot.com) platform.  
Receive SMS online with temporary US phone numbers — programmatic access for SMS verification, OTP codes, and virtual number management.

> **Live version of this document:** [tellabot.com/api_command_reference.php](https://www.tellabot.com/api_command_reference.php)

---

## Endpoint

All API requests are made to:

```
https://www.tellabot.com/api_command.php
```

Parameters are passed as a **GET query string**.

---

## Authentication

Every request requires two parameters:

| Parameter | Description |
|-----------|-------------|
| `user` | Your username or email |
| `api_key` | Your API key |

Generate your API key at **Account → Profile** inside the members area. This action requires email confirmation.

---

## Response format

All commands return a JSON object with two fields:

```json
{ "status": "ok",    "message": ... }
{ "status": "error", "message": "Error description" }
```

When `status` is `"ok"`, `message` contains the result data (array or value).  
When `status` is `"error"`, `message` contains a human-readable error description.

---

## One-time MDNs

### 1. Request an MDN

Request a temporary phone number for SMS verification.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `cmd` | Y | `"request"` |
| `user` | Y | Your username or email |
| `api_key` | Y | Your API key |
| `service` | Y | Service name, as returned by `list_services` or shown on **Billing → Services and Rates** |
| `mdn` | N | Request a specific phone number. Returns an error if the number is already in use or unavailable |
| `areacode` | N | 3-digit US area code. Takes precedence over `state`. Ignored if `mdn` is passed |
| `state` | N | 2-letter US state abbreviation. Ignored if `mdn` or `areacode` is passed |
| `markup` | N | Priority bid, integer 10–2000. See [Priority requests](#priority-requests) below |

**Response fields** (inside `message` array):

| Field | Description |
|-------|-------------|
| `id` | Request ID |
| `mdn` | Assigned phone number (empty if status is `Awaiting MDN`) |
| `service` | Service name |
| `status` | `Reserved` or `Awaiting MDN` |
| `state` | State code for geo-targeted requests |
| `markup` | Bid value |
| `price` | Price |
| `carrier` | Carrier name |
| `till_expiration` | Seconds until the request expires |

**Example request:**
```
https://www.tellabot.com/api_command.php?cmd=request&user=test&api_key=0123456789&service=Amazon
```

**Successful response:**
```json
{
  "status": "ok",
  "message": [
    {
      "id": "10000001",
      "mdn": "15302286946",
      "service": "Amazon",
      "status": "Reserved",
      "state": "",
      "markup": 0,
      "price": 0.50,
      "carrier": "TMobile",
      "till_expiration": 900
    }
  ]
}
```

**Error responses:**
```json
{ "status": "error", "message": "Invalid service name Goooooogle" }
{ "status": "error", "message": "No numbers available, retry later" }
```

#### Reusing an MDN

To reuse a number you previously used with a service, pass the `mdn` parameter:

```
https://www.tellabot.com/api_command.php?cmd=request&user=test&api_key=0123456789&service=Amazon&mdn=12345678901
```

Note: numbers are rotated periodically and may no longer be available.

#### Priority requests

Pass the `markup` parameter (10–2000) to place a priority bid when no numbers are immediately available.

- The request is created with status `Awaiting MDN` and the `mdn` field is empty.
- When a suitable number becomes available and multiple users have placed bids, it goes to the highest bidder.
- The status changes to `Reserved` once a number is assigned.
- An unfulfilled priority request is automatically deleted after 15 minutes.
- **Tip:** use `list_services` with a single service name to get `recommended_markup` as a starting point.

```json
{
  "status": "ok",
  "message": [
    {
      "id": "10000001",
      "mdn": "",
      "service": "Amazon",
      "status": "Awaiting MDN",
      "state": "CA",
      "markup": 25,
      "price": 0.60,
      "carrier": "",
      "till_expiration": 900
    }
  ]
}
```

Monitor the request with `request_status`, or configure a [webhook URL](#webhook-url) to receive a `priority_request` event when your bid wins.

#### Requesting the same MDN for multiple services at once

Pass up to 5 comma-separated service names in the `service` parameter. The system creates linked priority requests and assigns a single phone number that works for all listed services simultaneously.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `cmd` | Y | `"request"` |
| `user` | Y | Your username or email |
| `api_key` | Y | Your API key |
| `service` | Y | Up to 5 service names, comma-separated |
| `areacode` | N | 3-digit US area code |
| `state` | N | 2-letter US state abbreviation |

Markup is assigned automatically. Note: geo targeting reduces the pool of available numbers.

**Example request:**
```
https://www.tellabot.com/api_command.php?cmd=request&user=test&api_key=0123456789&service=Yahoo,Google,Amazon
```

**Example response:**
```json
{
  "status": "ok",
  "message": [
    {
      "id": "10000001",
      "mdn": "",
      "service": "Amazon",
      "status": "Awaiting MDN",
      "state": "",
      "markup": 10,
      "price": 0.30,
      "carrier": "",
      "till_expiration": 900
    },
    {
      "id": "10000002",
      "mdn": "",
      "service": "Google",
      "status": "Awaiting MDN",
      "state": "",
      "markup": 110,
      "price": 1.30,
      "carrier": "",
      "till_expiration": 900
    },
    {
      "id": "10000003",
      "mdn": "",
      "service": "Yahoo",
      "status": "Awaiting MDN",
      "state": "",
      "markup": 50,
      "price": 0.45,
      "carrier": "",
      "till_expiration": 900
    }
  ]
}
```

---

### 2. Get request status

Get the current status of a request by its ID.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `cmd` | Y | `"request_status"` |
| `user` | Y | Your username or email |
| `api_key` | Y | Your API key |
| `id` | Y | Request ID returned by the `request` command |

**Status values** (inside `message[0].status`):

| Value | Meaning |
|-------|---------|
| `Awaiting MDN` | Priority bid placed; no number assigned yet. `mdn` field is empty |
| `Reserved` | Number assigned, waiting for incoming SMS. `mdn` contains the number |
| `Completed` | SMS received. Use `read_sms` to retrieve the message |
| `Rejected` | Rejected by you via `reject`, or no suitable number was available |
| `Timed Out` | No SMS arrived within the allowed time; request was automatically cancelled |

> **Polling note:** wait at least 15 seconds between `request_status` calls.

**Example request:**
```
https://www.tellabot.com/api_command.php?cmd=request_status&user=test&api_key=0123456789&id=10000001
```

**Example responses:**
```json
{
  "status": "ok",
  "message": [
    {
      "id": "10000001",
      "mdn": "",
      "service": "Amazon",
      "status": "Awaiting MDN",
      "state": "CA",
      "markup": 20,
      "carrier": "",
      "till_expiration": 300
    }
  ]
}
```
```json
{
  "status": "ok",
  "message": [
    {
      "id": "10000001",
      "mdn": "12345678901",
      "service": "Amazon",
      "status": "Reserved",
      "state": "CA",
      "markup": 20,
      "carrier": "ATT",
      "till_expiration": 890
    }
  ]
}
```

**Error response:**
```json
{ "status": "error", "message": "Invalid request ID" }
```

---

### 3. Reject an MDN

Reject a reserved number. After rejection this number will not be offered to you again.  
Can also be used to cancel a priority bid (`Awaiting MDN` status).

| Parameter | Required | Description |
|-----------|----------|-------------|
| `cmd` | Y | `"reject"` |
| `user` | Y | Your username or email |
| `api_key` | Y | Your API key |
| `id` | Y | Request ID returned by the `request` command |

**Example request:**
```
https://www.tellabot.com/api_command.php?cmd=reject&user=test&api_key=0123456789&id=10000001
```

**Successful response:**
```json
{ "status": "ok", "message": "MDN has been rejected" }
```

**Error response:**
```json
{ "status": "error", "message": "Invalid request ID" }
```

---

### 4. Read SMS

Retrieve messages received to a phone number.

> **Note:** each one-time MDN request can only receive a single SMS. To receive another message for the same service, make a new `request` — optionally passing the same `mdn` to reuse the number.  
> **Note:** when filtering by `id`, this command only returns a result after the request status changes to `Completed`.  
> **Tip:** consider using a [webhook URL](#webhook-url) instead of polling.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `cmd` | Y | `"read_sms"`. Returns up to 3 latest messages from the past 2 days, newest first |
| `user` | Y | Your username or email |
| `api_key` | Y | Your API key |
| `id` | N | Filter by request ID. If passed, `mdn` and `service` are ignored |
| `mdn` | N | Filter by phone number |
| `service` | N | Filter by service name |

**Response fields** (inside `message` array):

| Field | Description |
|-------|-------------|
| `timestamp` | UNIX timestamp of receipt |
| `date_time` | Human-readable date/time (America/New_York timezone) |
| `from` | Sending number |
| `to` | Receiving number |
| `service` | Service name |
| `price` | Price |
| `reply` | Full SMS text |
| `pin` | Extracted PIN code (when recognized) |

**Example request:**
```
https://www.tellabot.com/api_command.php?cmd=read_sms&user=test&api_key=0123456789&service=Google
```

**Successful response:**
```json
{
  "status": "ok",
  "message": [
    {
      "timestamp": "1600108956",
      "date_time": "2020-09-14 14:42:36 EDT",
      "from": "22000",
      "to": "18503814729",
      "service": "Google",
      "price": 1.20,
      "reply": "G-804036 is your Google verification code.",
      "pin": "G-804036"
    }
  ]
}
```

**Error response:**
```json
{ "status": "error", "message": "No messages" }
```

---

## Balance & Other Info

### 1. List services

List available services and their prices.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `cmd` | Y | `"list_services"` |
| `user` | Y | Your username or email |
| `api_key` | Y | Your API key |
| `service` | N | One or more service names, comma-separated. If omitted, all services are listed |

**Response fields** (inside `message` array):

| Field | Description |
|-------|-------------|
| `name` | Service name |
| `price` | One-time SMS price |
| `ltr_price` | Long-term rental price (30 days) |
| `ltr_short_price` | Long-term rental price (3 days) |
| `otp_available` | Approximate number of available one-time numbers |
| `ltr_available` | Approximate number of available long-term numbers |
| `recommended_markup` | Suggested priority bid value (returned only when querying a single service) |

> **Note:** availability figures are approximate and not calculated in real time. Actual availability is confirmed only when you make a `request`.

**Example request:**
```
https://www.tellabot.com/api_command.php?cmd=list_services&user=test&api_key=0123456789&service=Google
```

**Successful response:**
```json
{
  "status": "ok",
  "message": [
    {
      "name": "Google",
      "price": "1.00",
      "ltr_price": "20.00",
      "ltr_short_price": "5.00",
      "otp_available": "74",
      "ltr_available": "3",
      "recommended_markup": "10"
    }
  ]
}
```

**Error response:**
```json
{ "status": "error", "message": "Invalid service name DummyService" }
```

---

### 2. View balance

| Parameter | Required | Description |
|-----------|----------|-------------|
| `cmd` | Y | `"balance"` |
| `user` | Y | Your username or email |
| `api_key` | Y | Your API key |

**Example request:**
```
https://www.tellabot.com/api_command.php?cmd=balance&user=test&api_key=0123456789
```

**Response:**
```json
{ "status": "ok", "message": "10.00" }
```

---

## Webhook URL

### Setup

Go to **Account → Profile** in the members area and enter your webhook URL.

- Data is sent as an **HTTP POST** request to your URL.
- Your endpoint **must return HTTP 200**. Redirects are not followed.
- If the URL is unavailable or returns a non-200 status, the system retries 5 more times at 10-minute intervals.

---

### Incoming message

Sent when an SMS is received to one of your numbers.

| Field | Value |
|-------|-------|
| `event` | `"incoming_message"` |
| `id` | Request ID |
| `timestamp` | UNIX timestamp |
| `date_time` | Human-readable date/time (America/New_York) |
| `from` | Sending number |
| `to` | Receiving number |
| `service` | Service name |
| `reply` | SMS text |
| `pin` | Extracted PIN code (when recognized) |
| `price` | Price |

---

### Priority request won

Sent when your priority bid wins and a number is assigned.

| Field | Value |
|-------|-------|
| `event` | `"priority_request"` |
| `status` | `"ok"` |
| `id` | Request ID |
| `mdn` | Assigned phone number |
| `service` | Service name |
| `price` | Price |

---

## Links

- [Tell A Bot](https://www.tellabot.com) — receive SMS online, temporary US phone numbers for SMS verification
- [Live API reference](https://www.tellabot.com/api_command_reference.php)
