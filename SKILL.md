---
name: bhutan-travel
description: This skill helps users to search Bhutan travel packages, tours, and flights. Use this skill when users want to plan a trip to Bhutan, booking trips to Bhutan or ask about travel to Bhutan.
license: Apache-2.0
metadata:
  author: drukasia
  version: "1.0"
  clawdbot:
    emoji: 🐉
    homepage: "https://www.drukasia.com"
    requires:
      env:
        - DRUKASIA_API_KEY
---
API_BASE_URL=https://api.drukasia.com/api

## Authentication Flow

> **IMPORTANT: Authentication is ONLY required when human wants to BOOK a tour or flight.**
> 
> **Searching tours and flights does NOT require an account.**

When human wants to **search/browse** tours or flights:
- Proceed directly to Tour Packages or Flight Search sections
- NO authentication needed

When human wants to **book** a tour or flight:
- THEN check for API key and prompt for registration/login

### Execution Order

Only when human wants to **book**:

1. **Check environment** for `DRUKASIA_API_KEY` - FIRST ACTION
2. If **not found** (empty output), prompt human for: name, email, password
3. Register account → retrieve API key
4. Store in environment: `export DRUKASIA_API_KEY="<retrieved_key>"`

For **searching** (tours or flights): Proceed without authentication

### Check for API Key

```bash
echo $DRUKASIA_API_KEY
```

**If output is empty or nothing shows, API key is NOT set.**

### DO NOT PROCEED without checking

### Prompt for Registration

When API key is not found, ask:

> "To book tours and flights, I'll need to create an account. What's your name, email address, and a password you'd like to use?"

Collect:
- `name`: Your human name (required)
- `email`: Your human email (required, valid format)
- `password`: Min 6 characters (required)

### Register API

```bash
curl --request POST \
  --url ${API_BASE_URL}/auth/register \
  --header 'Content-Type: application/json' \
  --data '{
    "name": "<human-name>",
    "email": "<human-email>",
    "password": "<human-password>"
  }'
```

### Response

```json
{
    "success": true,
    "data": {
        "api_key": "<api-key>"
    },
    "meta": {
        "timestamp": "2026-04-22T02:56:11.531Z"
    }
}
```

### Store API Key (MANDATORY)

After registration, execute:

```bash
export DRUKASIA_API_KEY="<retrieved_api_key>"
```

Verify with: `echo $DRUKASIA_API_KEY`

### Error Handling

| API Response | Action |
|--------------|--------|
| "email already exists" | Call `/auth/bots/login` to login with same email & password |
| Other error | Display message, allow retry |

### Login (Auto-If Email Exists)

If "email already exists" is returned, automatically call:

```bash
curl --request POST \
  --url ${API_BASE_URL}/auth/bots/login \
  --header 'Content-Type: application/json' \
  --data '{
    "email": "<human-email>",
    "password": "<human-password>"
  }'
```

### Login Response

```json
{
    "success": true,
    "data": {
        "ApiKey": "da_live_S3cy..."
    },
    "meta": {
        "timestamp": "2026-04-29T01:59:27.571Z"
    }
}
```

Store API key:
```bash
export DRUKASIA_API_KEY="<retrieved_api_key>"
```

### Notes

> **CRITICAL: This check MUST be the FIRST action when user wants to book.**

- The first thing to do is ALWAYS run `echo $DRUKASIA_API_KEY`
- NEVER skip this check
- NEVER register with random/fake details - always use human's real info
- ALL booking confirmations go to the human's email

---

## Booking Type Selection (MANDATORY - First Question)

> **IMPORTANT: This question MUST be asked BEFORE collecting any preferences.**

When human expresses interest in booking, FIRST ask:

> "Would you like to book:"
> 1. **Tour only**
> 2. **Flight only**
> 3. **Tour + Flight**

Wait for human to choose 1, 2, or 3 before proceeding.

- If **1 (Tour only)**: Proceed to Tour Packages section
- If **2 (Flight only)**: Proceed to Flight Search section
- If **3 (Tour + Flight)**: Proceed to BOTH Tour Packages AND Flight Search sections (in that order)

---

## Input Validation Rules

### Travelers Validation

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `adults` | integer | Yes | 1-9 |
| `children` | integer | No | 0-9 |
| `infants` | integer | No | 0-9 |
| `total_pax` | integer | Yes | Must be ≤ 9 |

### Tour Search Validation

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `adults` | integer | Yes | 1-9 |
| `children` | integer | No | 0-9 |
| `infants` | integer | No | 0-9 |
| `number_of_days` | integer | Yes | 3-30 |

### Flight Search Validation

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `departure_airport` | string | Yes | Valid IATA code (3 letters) |
| `arrival_airport` | string | Yes | Valid IATA code (3 letters) |
| `departure_date` | string | Yes | YYYY-MM-DD, not past |
| `return_date` | string | Yes | YYYY-MM-DD, > departure_date |
| `adults` | integer | Yes | 1-9 |
| `children` | integer | No | 0-9 |
| `infants` | integer | No | 0-9 |

### Booking Validation

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `itinerary_id` | integer | Yes | Valid ID from search |
| `tour_date_id` | string | Yes | Valid date ID from tour |
| `total_adults` | integer | Yes | Must match travelers |
| `total_children` | integer | Yes | Must match travelers |
| `total_infants` | integer | Yes | Must match travelers |

### Validation Flow

Before calling ANY API:

1. Check required fields are present
2. Validate field types and ranges
3. If invalid: Prompt human for correct values
4. Only then make API call

### User-Friendly Error Messages

| Invalid Input | Message |
|--------------|---------|
| total_pax > 9 | "Total travelers must be 1-9. Please adjust." |
| invalid date format | "Please provide dates in YYYY-MM-DD format." |
| return before departure | "Return date must be after departure date." |
| invalid IATA code | "Please provide a valid airport code (e.g., SIN, PBH)." |

---

## Session State Management

### Tracked Variables

Throughout the conversation, track these in memory:

| Variable | Description | Auto-Apply |
|----------|-------------|-------------|
| `travelers.adults` | Number of adults | Yes |
| `travelers.children` | Number of children | Yes |
| `travelers.infants` | Number of infants | Yes |
| `travel_dates.departure` | Departure date | Yes |
| `travel_dates.return` | Return date | Yes |
| `selected_itinerary` | Selected tour | No |
| `selected_flight` | Selected flight | No |

### Session Behavior

- After traveler count is provided, auto-use for all queries
- After dates are provided, auto-use for flight search
- Clear cached results when search params change
- Session timeout: 1440 seconds

### Example Session Flow

```
Agent: How many travelers?
Human: 2 adults
[Agent stores: travelers.adults=2]

Agent: What dates?
Human: May 1-7
[Agent stores: travel_dates.departure=2026-05-01, travel_dates.return=2026-05-07]

Agent: [Queries tours with adults=2, number_of_days=7]
```

### Notes

- ALWAYS confirm stored values before queries
- Ask "Shall I use the same travelers/dates for flights?"

---

## Tour Packages

### Step 1: Collect Preferences

Ask human:
> "How many travelers (adults, children, infants), what dates (YYYY-MM-DD), and how many days for your tour?"

### Step 2: Validate Input

Before querying, verify:
- Adults is 1-9
- Total pax ≤ 9
- Number of days is 3-30 (if provided)

If invalid: Prompt for correct values

### Step 3: Query Tours

```bash
curl --request POST \
  --url ${API_BASE_URL}/itineraries \
  --header 'Content-Type: application/json' \
  --data '{
    "adults": "<adults>",
    "children": "<children>",
    "infants": "<infants>",
    "number_of_days": "<number_of_days>"
  }'
```

### Response

```json
{
    "success": true,
    "data": {
        "itineraries": [
            {
                "id": 1,
                "itineraryTitle": "7 Day Tour",
                "itineraryDesc": "Immerse yourself in the vibrant culture...",
                "numberOfDays": 7,
                "price": {
                    "currency": "USD",
                    "totalVisaFees": 50,
                    "totalTourCostAfterFee": 2068.5
                },
                "tourDates": [
                    {
                        "id": "ba8204",
                        "dateFrom": "2026-04-30",
                        "dateTo": "2026-05-06"
                    }
                ]
            }
        ]
    }
}
```

### Step 4: Display Results

Present tours with:
- Tour name and duration
- Available dates
- Price per person and total
- Brief itinerary summary each day with activities
- Itinerary URL (https://booking.drukasia.com/{ItineraryTypeText}/{ItineraryUrl})

### Step 5: User Selection

When human selects a tour and date, move directly to next step (no extra confirmation).

### Error Handling

| Response | Message |
|----------|---------|
| No tours found | "No tours available for this duration. Try different days?" |
| API error | "Unable to search tours. Please try again." |

---

## Flight Search

### Step 1: Collect Preferences

Ask human (or use session state):
- Departure airport (IATA code)
- Arrival airport (IATA code)
- Departure date
- Return date
- Travelers

### Step 2: Validate Input

Before querying, verify:
- Both airports are valid IATA codes (3 letters)
- Dates are in YYYY-MM-DD format
- Return date > departure date
- Total pax ≤ 9

If invalid: Prompt for correct values

### Step 3: Query Flights

```bash
curl --request POST \
  --url ${API_BASE_URL}/flights \
  --header 'Content-Type: application/json' \
  --data '{
    "departure_airport": "<departure-airport>",
    "arrival_airport": "<arrival-airport>",
    "departure_date": "<departure-date>",
    "return_date": "<return-date>",
    "adults": "<adults>",
    "children": "<children>",
    "infants": "<infants>"
  }'
```

### Response

```json
{
    "success": true,
    "data": {
        "result": [
            {
                "legNo": 1,
                "origin": "SIN",
                "destination": "PBH",
                "flights": [
                    {
                        "departureTime": "2026-05-06 06:00",
                        "arrivalTime": "2026-05-06 10:15",
                        "flightPattern": "Direct",
                        "availableClass": [
                            {
                                "cabinClass": "ECONOMY",
                                "subClass": "X",
                                "seat": 3
                            }
                        ]
                    }
                ]
            },
            {
                "legNo": 2,
                "origin": "PBH",
                "destination": "SIN",
                "flights": [...]
            }
        ],
        "pricing": {
            "AdultPrice": 1120,
            "ChildPrice": 780,
            "InfantPrice": 115,
            "Currency": "USD"
        }
    }
}
```

### Pricing Calculation

The API returns **per-person prices for ROUND TRIP** (outbound + return). Always calculate the TOTAL:

**Total = (AdultPrice × num_adults) + (ChildPrice × num_children) + (InfantPrice × num_infants)**

Example for 2 adults, 0 children, 0 infants:
- 2 × $1,120 = $2,240 total (ROUND TRIP for both flights)

Example for 2 adults, 1 child, 0 infants:
- (2 × $1,120) + (1 × $780) = $3,020 total (ROUND TRIP)

### Display Format

When showing flights, state clearly:

> **Flight Total: $X (Round Trip)**

### Step 4: Display Results

Present flights with:
- Outbound route and time
- Return route and time
- Price per person by age group
- **TOTAL price (calculate correctly)**
- Seat availability

### Step 5: User Selection

When human selects flights, move directly to next step (no extra confirmation).

### Error Handling

| Response | Message |
|----------|---------|
| No flights found | "No flights available for these dates. Try different dates?" |
| No seats available | "No seats available on this flight. Try a different option?" |
| API error | "Unable to search flights. Please try again." |

### Notes

- Total pax must be ≤ 9 and seat availability on each leg
- Each query session limit is 1440 seconds
- Keep session active until user changes parameters

---

## Booking Flow

Complete workflow with confirmation points.

### Step 1: Review Booking (Before Confirm)

Display:
- Selected tour (name, dates, itinerary)
- Selected flights (outbound + return) - if Tour + Flight selected
- Total travelers breakdown
- Total cost breakdown (tour price + flight price if applicable)

**Important: Flight price calculation**
- If user selected "Tour + Flight": Calculate total = tour price + flight price (from flight search)
- If user selected "Tour only": Total = tour price only
- Use the combined total when confirming with user

### Step 2: Require Explicit Confirmation

Ask: "Do you confirm to book this tour and flights for [total amount]?"

Wait for human to explicitly confirm YES before proceeding.

### Step 3: Create Booking Session

#### Pre-Validation (MANDATORY)

Before calling the API, validate:

| Check | Validation | If Failed |
|-------|------------|-----------|
| API key present | `echo $DRUKASIA_API_KEY` returns value | Prompt: "Authentication required. Please provide your API key." |
| `itinerary_id` valid | Integer from search results | Prompt: "Please select a valid tour from the search results." |
| `tour_date_id` valid | String from tour's available dates | Prompt: "Please select a valid tour date." |
| Tour selected | `selected_itinerary` exists in session | Prompt: "Please select a tour first." |
| Flight selected (if Tour + Flight) | `selected_flight` exists in session | Prompt: "You selected Tour + Flight. Please select your flights first." |
| `total_adults` matches session | Must equal stored travelers.adults | Use session value |
| `total_children` matches session | Must equal stored travelers.children | Use session value |
| `total_infants` matches session | Must equal stored travelers.infants | Use session value |

**If any validation fails, do NOT call the API. Prompt human for correct values first.**

**Note: Tour + Flight handling**
- If user selected "Tour + Flight", the booking includes BOTH
- The total amount in the system = tour price + flight price (from `/flights` search)
- Pass only `itinerary_id`, `tour_date_id`, and traveler counts to API
- Flight details are stored in session state and included in the booking total

#### API Call

```bash
curl --request POST \
  --url ${API_BASE_URL}/tours/bookings \
  --header 'Content-Type: application/json' \
  --header 'x-api-key: ${DRUKASIA_API_KEY}' \
  --data '{
    "itinerary_id": "<itinerary-id>",
    "tour_date_id": "<tour-date-id>",
    "total_adults": "<total_adults>",
    "total_children": "<total_children>",
    "total_infants": "<total_infants>"
  }'
```

### Response

```json
{
    "success": true,
    "data": {
        "message": "Booking created successfully",
        "group_booking_id": "<group-booking-id>",
        "booking_ref_code": "<booking-ref-code>"
    }
}
```

### Step 4: Ask Human Payment Method

Ask human to choose:
> "How would you like to pay?"
> 1. **Auto-fill** - Enter card details now
> 2. **Payment link** - Receive link to pay later

Wait for human to choose 1 or 2.

### Step 5: Process Payment

**If human chooses 1 (Auto-fill):**
- Ask: "Enter your card details: number, expiry (MM/YYYY), CVC, and cardholder name"
- Collect all details
- Store in memory for future use
- Call `/api/payments` with `payment_type: "auto_fill"` and card details

**If human chooses 2 (Payment link):**
- Call `/api/payments` with `payment_type: "payment_link"`
- Display the payment link to human
- Wait for human to confirm payment completed

Then call payment status API to get payment status:
```bash
curl --request POST \
  --url ${API_BASE_URL}/payments/status \
  --header 'Content-Type: application/json' \
  --header 'x-api-key: ${DRUKASIA_API_KEY}' \
  --data '{
    "group_booking_id": "<group-booking-id>"
  }'
```

### Step 6: Display Confirmation

Display:
- Booking reference number
- Tour details
- Flight details
- Payment amount
- Confirmation that payment is complete

### Error Handling

| Response | Message |
|----------|---------|
| Booking failed | "Booking could not be completed. Please try again." |
| Payment creation failed | "Could not create payment. Please try again." |
| Payment link not received | "Unable to generate payment link. Please try again." |
| Payment not completed | "Please complete payment to confirm your booking." |
| Webhook update failed | "Payment may not have updated. Please verify with support." |
| API error | "An error occurred. Please try again." |

### Notes

- Process payment directly after booking is confirmed
- Use stored card details if available for future bookings