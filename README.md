# Kai Ika – Weekly Can Pickup Web App

This project is a lightweight web app used to manage weekly can pickups from partner sites.
It consists of:

* A **static HTML frontend** (hosted on GitHub Pages)
* A **Google Apps Script backend** (public Web App)
* A **Google Sheet** as the database

No authentication is required — Apps Script acts as a secure bridge between the public frontend and the private Google Sheet.

---

## High-level architecture

```
Browser (HTML / JS)
        ↓
Apps Script Web App (doGet / doPost)
        ↓
Google Sheet (Sites, Pickup Log)
```

The browser **never talks directly to Google Sheets**.

---

## Google Sheet structure

### 1. `Sites` sheet (master list)

This sheet defines *what* sites exist and *how often* they should be picked up.

| Column | Name             | Description                              |
| ------ | ---------------- | ---------------------------------------- |
| A      | Site ID          | Unique identifier (string or number)     |
| B      | Site name        | Display name                             |
| C      | Address          | Optional                                 |
| D      | Pickup frequency | `Weekly`, `Fortnightly`, or `Monthly`    |
| E      | Receptacle       | Optional default                         |
| F      | Number of bins   | Optional default                         |
| G      | Active           | **Must be TRUE** for site to be included |

Only rows where **Active = TRUE** are considered.

---

### 2. `Pickup Log` sheet (history)

This sheet stores all completed pickups.
Rows are **append-only**.

| Column | Name           |
| ------ | -------------- |
| A      | Timestamp      |
| B      | Pickup date    |
| C      | Site ID        |
| D      | Site name      |
| E      | Receptacle     |
| F      | Number of bins |
| G      | Fill quantity  |

This sheet is used to determine the **last pickup date** for each site.

---

## How the weekly pickup schedule works

### Step 1: Frontend request

When the user clicks **“Load This Week”**, the frontend calls:

```
GET ?action=getWeeklySchedule
```

This hits the Apps Script `doGet()` function.

---

### Step 2: Apps Script loads data

Apps Script explicitly opens the spreadsheet by ID:

```js
SpreadsheetApp.openById(SPREADSHEET_ID)
```

It then:

* Reads all rows from the `Sites` sheet
* Reads all rows from the `Pickup Log` sheet

---

### Step 3: Determine last pickup per site

For each site:

* All pickup log rows matching that **Site ID** are found
* The most recent `Pickup date` is selected
* If no pickup exists, the site is treated as **never collected**

---

### Step 4: Calculate next due date

Based on `Pickup frequency`:

| Frequency   | Rule     |
| ----------- | -------- |
| Weekly      | +7 days  |
| Fortnightly | +14 days |
| Monthly     | +1 month |

Logic:

* If the site has been picked up before → next due = last pickup + interval
* If the site has **never** been picked up → next due = today

---

### Step 5: Determine “this week”

“This week” is defined as:

* **Monday 00:00 → Sunday 23:59**
* Uses the spreadsheet’s timezone

Apps Script calculates:

```js
weekStart
weekEnd
```

---

### Step 6: Filter sites due this week

A site is included **only if**:

```js
nextDue >= weekStart && nextDue <= weekEnd
```

If no sites match, an empty array (`[]`) is returned.

---

### Step 7: Frontend rendering

The frontend receives JSON like:

```json
[
  {
    "siteId": "1",
    "siteName": "Ben",
    "address": "",
    "frequency": "Weekly"
  }
]
```

For each site:

* A row is rendered
* Input fields are shown for:

  * Receptacle (dropdown)
  * Number of bins
  * Fill quantity

If the array is empty, the UI displays:

> “No pickups due this week.”

---

## Submitting pickup data

When the user clicks **“Submit Pickup Data”**:

1. The frontend gathers all filled rows
2. Sends them via:

   ```
   POST ?action=submitPickupData
   ```
3. Apps Script appends rows to the `Pickup Log` sheet

Only rows with a selected **receptacle** are saved.

---

## Common troubleshooting

**No pickups loading?**

* Check `Active = TRUE` (boolean, not text)
* Check `Pickup frequency` spelling
* Ensure Apps Script is redeployed
* Ensure Apps Script uses `openById()`, not `getActive()`
