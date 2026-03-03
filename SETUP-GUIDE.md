# Fabricly — Etsy to Shipmondo via Make.com
# Complete Build Guide

---

## BEFORE YOU START — Gather These Credentials

1. **Shipmondo API**
   - Log in to Shipmondo → Settings → API Access
   - Copy: `Username` and `API Key`
   - Base URL: `https://app.shipmondo.com/api/public/v3`

2. **Anthropic API** (for Claude AI fabric classification)
   - Go to: https://console.anthropic.com
   - Create an API key
   - Save it somewhere safe — you'll paste it into Make.com

3. **Google Sheets**
   - Upload `hs-code-lookup.csv` (from this folder) to Google Sheets
   - **Rename the sheet tab** (bottom of screen) to exactly: `HS_Lookup`
   - Note the spreadsheet ID from the URL:
     `https://docs.google.com/spreadsheets/d/SPREADSHEET_ID_HERE/edit`

---

## PHASE 1 — Google Sheets Setup

1. Open Google Drive
2. Upload `hs-code-lookup.csv` — it will auto-convert to a Google Sheet
3. Click the sheet tab at the bottom → Rename to `HS_Lookup`
4. Verify structure:
   - Column A = Keyword (e.g. "Cotton Jersey")
   - Column B = Fabric Description (e.g. "Knitted fabric of cotton")
   - Column C = HS Code (e.g. "6006.22")
5. Done — leave this sheet open, you'll connect it in Make.com

---

## PHASE 2 — Build the Make.com Scenario

### Step 1: Create the Scenario

1. Log in to make.com
2. Click **"+ Create a new scenario"**
3. Name it: `Fabricly — Etsy to Shipmondo`

### Step 2: Add Etsy Trigger (Module 1)

1. Click the empty circle → search **"Etsy"** → select it
2. Choose trigger: **"Watch Receipts"**
3. Click **"Add"** to connect your Etsy account
   - A popup opens — log in with your Etsy seller account
   - Authorize Make.com access
4. Configure the module:
   - **Status**: `paid`
   - **Limit**: `10`
5. Click **OK**

### Step 3: Add Iterator (Module 2)

This handles multi-item orders by looping through each line item.

1. Click **+** after Etsy module
2. Search **"Flow Control"** → select **"Iterator"**
3. Set **Array** to: `{{1.Transactions}}`
4. Click **OK**

> The Iterator outputs one transaction at a time. All modules after this
> will reference `{{2.field}}` for the current line item and `{{1.field}}`
> for the original Etsy receipt data.

### Step 4: Add Router (Module 3)

1. Click **+** after the Iterator → select **"Router"** (under Flow Control)
2. Two paths appear automatically

**Configure Path 1 filter (EU Orders):**
1. Click the wrench icon on the top path line
2. Label: `EU Orders`
3. Condition:
   - Field: `{{1.shipping_address.country_iso}}`
   - Operator: **"Does not match pattern (case insensitive)"**
   - Value: `^(US|GB|CA|AU|NO|CH|JP|NZ|SG|KR|HK|TW|AE|SA|IL|TH|BR|MX|IN)$`
4. Click **OK**

**Configure Path 2 filter (Non-EU Orders):**
1. Click the wrench icon on the bottom path line
2. Label: `Non-EU Orders (Customs)`
3. Condition:
   - Field: `{{1.shipping_address.country_iso}}`
   - Operator: **"Matches pattern (case insensitive)"**
   - Value: `^(US|GB|CA|AU|NO|CH|JP|NZ|SG|KR|HK|TW|AE|SA|IL|TH|BR|MX|IN)$`
4. Click **OK**

---

### Step 5: Path 1 — EU Order → Shipmondo (Module 4)

1. On Path 1, click **+** → search **"HTTP"** → select **"Make a request"**
2. Configure:

| Setting         | Value                                                   |
|-----------------|---------------------------------------------------------|
| URL             | `https://app.shipmondo.com/api/public/v3/orders`        |
| Method          | `POST`                                                  |
| Headers         | `Content-Type` : `application/json`                     |
| Authentication  | `Basic Auth`                                            |
| Username        | Your Shipmondo API username                             |
| Password        | Your Shipmondo API key                                  |
| Body type       | `Raw`                                                   |
| Content type    | `JSON (application/json)`                               |

3. In the **Body** field, paste the contents of `shipmondo-eu-order.json`
4. Click **OK**

> **Module number note:** The `{{1.xxx}}` references point to Module 1 (Etsy trigger).
> The `{{2.xxx}}` references point to Module 2 (Iterator output = current line item).
> Make.com shows module numbers at the top of each module circle.

---

### Step 6: Path 2 — Claude AI Fabric Classification (Module 5)

1. On Path 2, click **+** → search **"HTTP"** → select **"Make a request"**
2. Configure:

| Setting         | Value                                                   |
|-----------------|---------------------------------------------------------|
| URL             | `https://api.anthropic.com/v1/messages`                 |
| Method          | `POST`                                                  |
| Headers         | `Content-Type` : `application/json`                     |
|                 | `x-api-key` : `YOUR_ANTHROPIC_API_KEY`                  |
|                 | `anthropic-version` : `2023-06-01`                      |
| Body type       | `Raw`                                                   |
| Content type    | `JSON (application/json)`                               |

3. In the **Body** field, paste the contents of `claude-ai-request.json`
4. Click **OK**

> The Claude response returns the keyword at: `{{5.data.content[1].text}}`
> (Module 5 output → data → content array → first item → text field)
> In Make.com you may need to map it as: `{{5.data.content[].text}}`

---

### Step 7: Google Sheets HS Code Lookup (Module 6)

1. Click **+** after Claude module → search **"Google Sheets"**
2. Select **"Search Rows"**
3. Configure:

| Setting         | Value                                                   |
|-----------------|---------------------------------------------------------|
| Connection      | Click Add → authorize your Google account               |
| Spreadsheet     | Select your HS Code Lookup spreadsheet                  |
| Sheet           | `HS_Lookup`                                             |
| Column to search| `A` (Keyword column)                                    |
| Search value    | `{{trim(5.data.content[].text)}}`                       |

4. Click **OK**

> **trim()** removes any accidental whitespace from the Claude response.
> The output gives you:
> - `{{6.A}}` = Keyword
> - `{{6.B}}` = Fabric Description
> - `{{6.C}}` = HS Code

---

### Step 8: Non-EU Order → Shipmondo with Customs (Module 7)

1. Click **+** after Google Sheets → search **"HTTP"** → **"Make a request"**
2. Same Shipmondo auth settings as Module 4 (Step 5)
3. In the **Body** field, paste the contents of `shipmondo-non-eu-order.json`

**IMPORTANT — Update these references in the JSON:**
   - Replace `{{5.B}}` with `{{6.B}}` (fabric description from Google Sheets)
   - Replace `{{5.C}}` with `{{6.C}}` (HS code from Google Sheets)
   - The `replace()` function strips dots: `{{replace(6.C; "."; "")}}`
   - Replace `YOUR_ADDRESS_LINE_1`, `YOUR_ZIPCODE`, `YOUR_CITY`, `YOUR_CVR_NUMBER`
     with your actual Fabricly business address and CVR number

4. Click **OK**

---

## PHASE 3 — Testing

### Test 1: EU Order
1. Click **"Run once"** in Make.com
2. Select a recent Etsy order shipped to an EU country (e.g. Germany, France)
3. All modules on Path 1 should show green checkmarks
4. Check Shipmondo → Orders → verify the order appears with correct details

### Test 2: Non-EU Order
1. Click **"Run once"** again
2. Select an order shipped to US, UK, or Norway
3. All modules on Path 2 should show green checkmarks
4. Check Shipmondo → Orders → verify:
   - Order appears
   - Customs section is populated
   - HS code is correct (no dots)
   - Fabric description matches

### Test 3: Multi-Item Order
1. Find or create a test order with 2+ items
2. Run once — the Iterator should fire twice
3. Verify both items appear in Shipmondo

### Common Errors

| Error                              | Fix                                        |
|------------------------------------|--------------------------------------------|
| 401 Unauthorized (Shipmondo)       | Check API username/key in Basic Auth        |
| 401 Unauthorized (Claude)          | Check x-api-key header value               |
| 404 Not Found                      | Verify URL is exactly right (no typos)      |
| Empty HS code                      | Check Google Sheets tab name = `HS_Lookup`  |
| HS code has dots (rejected)        | Add `replace()` function around HS code     |
| Only first item processed          | Make sure Iterator is before the Router     |
| "country_iso" is empty             | Use `country_code` field instead            |

---

## PHASE 4 — Go Live

1. Turn on the scenario toggle (bottom-left of scenario editor)
2. Set schedule:
   - **Core plan**: Every 1 minute (recommended)
   - **Free plan**: Every 15 minutes
3. Under Scenario settings (gear icon):
   - **Max number of cycles**: 1
   - **Sequential processing**: ON (prevents duplicate orders)

---

## PHASE 5 — Error Handling (Recommended)

Add these after going live:

### Option A: Email on Error
1. Right-click any HTTP module → **"Add error handler"**
2. Add **"Tools → Send email"** (or use Gmail/Slack module)
3. Configure to send yourself an alert with `{{error.message}}`

### Option B: Make.com Built-in Notifications
1. Go to scenario settings (gear icon)
2. Enable **"Notify me when scenario fails"**
3. You'll get an email from Make.com on any module failure

### Option C: Break + Retry (Recommended for HTTP modules)
1. Right-click the Shipmondo HTTP module → **"Add error handler"**
2. Select **"Break"** (under Error Handling)
3. Configure:
   - **Retries**: `3`
   - **Interval**: `60` seconds
4. This auto-retries failed API calls (handles temporary network issues)

---

## File Reference

| File                            | Purpose                                     |
|---------------------------------|---------------------------------------------|
| `hs-code-lookup.csv`           | Upload to Google Sheets as your HS lookup   |
| `shipmondo-eu-order.json`      | Paste into Path 1 HTTP module body          |
| `shipmondo-non-eu-order.json`  | Paste into Path 2 HTTP module body          |
| `claude-ai-request.json`       | Paste into Claude AI HTTP module body       |
| `regex-fallback-pattern.txt`   | Alternative to Claude AI (regex approach)   |
| `non-eu-countries.txt`         | Reference for Router filter country codes   |

---

## Scenario Flow Diagram

```
[Etsy: Watch Receipts]
        |
   [Iterator]  ← loops through each line item
        |
     [Router]
      /    \
     /      \
  [EU]    [Non-EU]
    |        |
    |    [Claude AI] → extract fabric keyword
    |        |
    |    [Google Sheets] → lookup HS code
    |        |
    |    [Shipmondo HTTP]  ← with customs block
    |
 [Shipmondo HTTP]  ← basic order, no customs
```
