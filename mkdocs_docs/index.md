# Tell A Bot - API Reference

Programmatic access to the [Tell A Bot](https://www.tellabot.com) platform.  
Request temporary US phone numbers, receive SMS messages, and retrieve OTP codes - all via a simple HTTP API.

> **Official live reference:** [tellabot.com/api_command_reference.php](https://www.tellabot.com/api_command_reference.php)

---

## Base URL

Send all API requests to:

```
https://www.tellabot.com/api_command.php
```

Pass parameters as a GET query string or a POST request.

---

## Authentication

Include these parameters in every request:

| Parameter | Description |
|-----------|-------------|
| `user` | Your username or email address |
| `api_key` | Your API key |

To generate an API key, go to **Account → Profile** in the members area. Email confirmation is required.

---

## Responses

Every command returns a JSON object with two fields:

```json
{ "status": "ok",    "message": ... }
{ "status": "error", "message": "Error description" }
```

On success, `message` holds the result.  
On failure, `message` contains a human-readable description of the error.

---

## One-time MDNs

### 1. Request an MDN

!!! note
    Each one-time MDN request can only receive a single SMS. To receive another message for the same number and service, submit a new request and pass the `mdn` parameter to reuse the same phone number.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `cmd` | Y | `"request"` |
| `user` | Y | Your username or email |
| `api_key` | Y | Your API key |
| `service` | Y | Service name, as returned by `list_services` or shown on **Billing → Services and Rates** |
| `mdn` | N | Request a specific phone number. Returns an error if the number is already in use or unavailable |
| `areacode` | N | Valid 3-digit US area code. Takes precedence over `state`. Ignored if `mdn` is passed |
| `state` | N | Valid 2-letter US state abbreviation. Ignored if `mdn` or `areacode` is passed |
| `markup` | N | Priority bid, integer 10–2000. See [Priority requests](#priority-requests) below |

**Response fields** (inside the `message` array):

| Field | Description |
|-------|-------------|
| `id` | Request ID |
| `mdn` | Assigned phone number (empty when status is `Awaiting MDN`) |
| `service` | Service name |
| `status` | `Reserved` or `Awaiting MDN` |
| `state` | State for geo-targeted requests |
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

#### Priority requests

Pass `markup` (10–2000) to place a priority bid when no numbers are immediately available.

- The request is created with status `Awaiting MDN` and `mdn` is empty.
- When a number becomes available and multiple users have placed bids, it goes to the highest bidder who bid earliest.
- The status changes to `Reserved` once a number is assigned.
- An unfulfilled priority request is automatically deleted after 15 minutes.
- Poll with `request_status`, or set up a [webhook URL](#webhook-url) to receive a `priority_request` event when your bid wins.
- **Tip:** call `list_services` with a single service name to get `recommended_markup` as a starting point.

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

#### Reusing an MDN

Pass the `mdn` parameter to request a number you have used before with the same service. Note that numbers rotate periodically and may no longer be available.

```
https://www.tellabot.com/api_command.php?cmd=request&user=test&api_key=0123456789&service=Amazon&mdn=12345678901
```

**Error response:**
```json
{ "status": "error", "message": "The MDN is not available" }
```

---

#### Requesting the same MDN for multiple services at once

Pass up to 5 comma-separated service names in `service` to obtain a single phone number valid for all of them simultaneously. Priority bids are created automatically with markup values high enough to secure the top position for every service.

!!! note
    Geo targeting may significantly reduce the pool of available numbers.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `cmd` | Y | `"request"` |
| `user` | Y | Your username or email |
| `api_key` | Y | Your API key |
| `service` | Y | Up to 5 service names, comma-separated |
| `areacode` | N | Valid 3-digit US area code. Takes precedence over `state` |
| `state` | N | Valid 2-letter US state abbreviation. Ignored if `areacode` is passed |

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

### 2. Check request status

Retrieve the current status of a request by its ID.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `cmd` | Y | `"request_status"` |
| `user` | Y | Your username or email |
| `api_key` | Y | Your API key |
| `id` | Y | Request ID returned by the `request` command |

**Response** - `message` array entries contain: `id`, `mdn`, `service`, `status`, `state`, `markup`, `carrier`, `till_expiration`.

**Status values:**

| Value | Meaning |
|-------|---------|
| `Awaiting MDN` | Priority bid placed; no number assigned yet. `mdn` is empty |
| `Reserved` | Number assigned, waiting for incoming SMS. `mdn` contains the number |
| `Completed` | SMS received. Use `read_sms` to retrieve the message |
| `Rejected` | Rejected via the `reject` command, or no suitable number was available |
| `Timed Out` | No SMS arrived in time; request was automatically cancelled |

!!! note
    Wait at least 15 seconds between `request_status` calls.

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

Reject a reserved number - it will not be offered to you again.  
Also works to cancel a priority bid with status `Awaiting MDN`.

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

Retrieve messages received to a phone number. Returns up to 3 of the latest messages from the past 2 days, newest first.

!!! note
    When filtering by `id`, results are only returned after the request status changes to `Completed`. Without `id`, messages from previously completed requests may also be included.  
    **Tip:** consider setting up a [webhook URL](#webhook-url) instead of polling.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `cmd` | Y | `"read_sms"` |
| `user` | Y | Your username or email |
| `api_key` | Y | Your API key |
| `id` | N | Filter by request ID. If passed, `mdn` and `service` are ignored |
| `mdn` | N | Filter by phone number |
| `service` | N | Filter by service name |

**Response fields** (inside the `message` array):

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
    },
    {
      "timestamp": "1600108852",
      "date_time": "2020-09-14 14:40:52 EDT",
      "from": "18339020112",
      "to": "15182193312",
      "service": "Google",
      "price": 1.20,
      "reply": "G-551858 is your Google verification code.",
      "pin": "G-551858"
    }
  ]
}
```

**Error response:**
```json
{ "status": "error", "message": "No messages" }
```

---

## Account Info

### 1. List services

Query available services along with their pricing and availability.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `cmd` | Y | `"list_services"` |
| `user` | Y | Your username or email |
| `api_key` | Y | Your API key |
| `service` | N | One or more service names, comma-separated. Omit to list all services |

**Response fields** (inside the `message` array):

| Field | Description |
|-------|-------------|
| `name` | Service name |
| `price` | One-time SMS price |
| `ltr_price` | (deprecated) Long-term rental price (30 days) |
| `ltr_short_price` | (deprecated) Long-term rental price (3 days) |
| `otp_available` | Approximate number of available one-time numbers |
| `ltr_available` | (deprecated) Approximate number of available long-term numbers |
| `recommended_markup` | Suggested priority bid (returned only when querying a single service) |

!!! note
    Availability figures are approximate and not real-time. Actual availability is confirmed only when you make a `request`.

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

### 2. Check balance

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

### Configuration

Set your webhook URL under **Account → Profile** in the members area.

- Payloads are delivered as **HTTP POST** requests.
- Your endpoint **must return HTTP 200**. Redirects are not followed.
- On failure, delivery is retried 5 more times at 10-minute intervals.

---

### Incoming message event

Fired when an SMS is delivered to one of your numbers.

| Field | Value |
|-------|-------|
| `event` | `"incoming_message"` |
| `id` | Request ID, as obtained with the `request` command |
| `timestamp` | UNIX timestamp |
| `date_time` | Human-readable date/time (America/New_York) |
| `from` | Sending number |
| `to` | Receiving number |
| `service` | Service name |
| `reply` | SMS text |
| `pin` | Extracted PIN code (when recognized) |
| `price` | Price |

---

### Priority request won event

Fired when your priority bid wins and a phone number is assigned.

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

- [Tell A Bot](https://www.tellabot.com) - receive SMS online, temporary US phone numbers for SMS verification
- [Live API reference](https://www.tellabot.com/api_command_reference.php)
- [Free US Numbers](https://www.tellabot.com/free-sms/) - free temporary numbers for testing the service
