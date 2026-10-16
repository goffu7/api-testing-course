# 🍜 Auntie Som's Noodle Stall – API Audit Report

Bruno test collection that audits Nephew Lek's noodle stall API.
The collection is in [`auntie-som-noodle-api-test/`](auntie-som-noodle-api-test). Saved run results are in [`test-results/`](test-results).

> **Result on a fresh server: 52 / 95 tests pass, 43 fail, 2 requests skipped** (the slow, opt-in token-expiry test; see below).
> **All 13 bugs found are covered by automated tests.** The token-expiry test is opt-in because it takes about 1 hour. Every failing test points to a real bug listed below. That is the expected result: the tests are written to **fail when the business logic is wrong**, even when the API answers `200 OK`.

---

## ▶️ How to run

```bash
# 1. start a FRESH server (stock and orders are kept in memory)
cd auntie-som-noodle-api
pip install flask
python app.py

# 2. in another terminal, run the whole collection with the Bruno CLI
cd auntie-som-noodle-api-test
npx @usebruno/cli run --env local -r

# 3. (optional, takes ~61 minutes) run the token-expiry test
npx @usebruno/cli run "06 Token Expiry (opt-in, slow)" --env local --env-var runExpiryTest=true
```

You can also open the `auntie-som-noodle-api-test` folder in the Bruno app (**Open Collection**), select the `local` environment, and run the collection with the Runner.

> ⚠️ Restart `python app.py` before every full run. Several requests change the stock, so a second run on the same server starts from a different state.

### Environment (`environments/local.bru`)

| Variable        | Value                   | Notes                                                       |
|-----------------|-------------------------|-------------------------------------------------------------|
| `baseUrl`       | `http://localhost:5000` | used by every request                                       |
| `username`      | `student`               | login body                                                  |
| `password`      | `hungry`                | login body                                                  |
| `accessToken`   | *(set by script)*       | secret var; saved by the login post-response script         |
| `runExpiryTest` | `false`                 | set to `true` to run the 1-hour token-expiry test (folder 06) |

---

## 🗂️ Collection structure

| Folder | What it checks |
|---|---|
| **01 Auth** | Valid login returns a UUID `accessToken` and `expiresIn: 3600`. A **post-response script saves the token** to the environment (`bru.setEnvVar`). A wrong password, missing fields or a `null` body must return 401/400 with no token, and must not crash the server. |
| **02 Happy Path Order** | Reads the menu (saves price and stock), places an order with `Bearer {{accessToken}}`, checks `totalPrice = price × quantity`, gets the order back by `{{orderId}}`, and checks the stock went down by exactly 1. |
| **03 Order Auth Checks** | No token, a fake token, or a token without the `Bearer ` prefix → 401. Unknown item → 404. A `null` body → 400 (no crash). None of them create an order. |
| **04 Stock and Quantity Rules** | Over-ordering, zero, negative, missing, text and fractional (`0.5`) quantities, buying the last stock, ordering a sold-out item. Also checks that stock is never negative and is always a whole number, and that error responses are JSON with no leaked stack trace. |
| **05 Get Order Edge Cases** | A non-existent order must return 404. An existing order must still be stored. Reading an order without a token must return 401. |
| **06 Token Expiry (opt-in, slow)** | Logs in, waits `expiresIn` + 5 seconds, then uses the old token. It must be rejected with 401. The folder is skipped unless `runExpiryTest=true`, because it takes about 1 hour. |

Values are passed between requests with scripts (`bru.setVar` / `bru.getVar`), not hard-coded. For example, the expected price comes from `GET /menu`, and the over-order quantity is `stock + 1`.

---

## 🐞 Bugs found (all covered by automated tests)

| # | Severity | Bug | Where (`app.py`) | Detected by (test → failure message) |
|---|---|---|---|---|
| 1 | 🔴 High | **Wrong price: every order is 5 baht short.** `totalPrice = price * quantity - 5` (a hidden discount). 1 × Tom Yum (50) returns **45**. | `create_order` | `Create Order - Valid` → *totalPrice equals menu price x quantity* (`expected 45 to equal 50`); `Get Order - By Id`; `Create Order - Buy All Remaining Stock` |
| 2 | 🔴 High | **You can order more than the stock, and stock goes negative.** The code only checks `stock <= 0`, not `stock >= quantity`. Ordering 2 Tom Yum when 1 is left returns `200 OK`, and stock becomes **-1**. | `create_order` | `Create Order - Quantity Exceeds Stock` (`expected 200 to equal 400`); `Get Menu - Stock Never Negative` (`Tom Yum Noodles stock: expected -1 to be at least +0`) |
| 3 | 🔴 High | **Zero quantity is accepted.** It creates a real order with `totalPrice: -5`, so Auntie *owes* the customer money. | `create_order` (no quantity validation) | `Create Order - Zero Quantity` (`expected -5 to be above +0`) |
| 4 | 🔴 High | **Negative quantity is accepted.** `quantity: -1` creates an order with `totalPrice: -50` and **adds** 1 to the stock. Customers could "sell" noodles back and create fake stock. | `create_order` | `Create Order - Negative Quantity` (`expected -50 to be above +0`); `Get Menu - Read Item B` (stock changed) |
| 5 | 🔴 High | **Fractional quantity is accepted.** `quantity: 0.5` creates an order for half a bowl (`totalPrice: 17.5`), and stock becomes **1.5**. | `create_order` (no integer check) | `Create Order - Fractional Quantity` (`expected 200 to equal 400`); `Get Menu - Read Item B` → *every stock value is a whole number* (`Fried Rice stock = 1.5`) |
| 6 | 🔴 High | **Tokens never expire.** Login says `expiresIn: 3600`, but `require_auth()` never checks `createdAt`, so a stolen token works forever (until the server restarts). | `require_auth` | `06 Token Expiry` → `Create Order - Expired Token` → *expired token is rejected with 401*. On this API the test should fail with `expected 404 to equal 401`, because the old token is still accepted. ⚠️ The full 61-minute run has not been completed: my run was stopped part-way when the machine ran low on memory. The bug is confirmed by code review (`createdAt` is stored but never read). |
| 7 | 🟠 Medium | **Order details are public.** `GET /orders/<id>` has no auth check, so anyone who knows or guesses an order id can read it. | `get_order` | `Get Order - Without Token` (`expected 200 to equal 401`) |
| 8 | 🟠 Medium | **The `Bearer ` prefix is not required.** `require_auth()` only does `replace("Bearer ", "")`, so `Authorization: <token>` without the scheme is also accepted. | `require_auth` | `Create Order - Token Without Bearer Prefix` (`expected 404 to equal 401`, because auth passed and the request reached the item lookup) |
| 9 | 🟠 Medium | **Missing quantity crashes the server (500).** `stock -= None` raises a `TypeError`. | `create_order` | `Create Order - Missing Quantity` → *server does not crash (no 5xx)* (`expected 500 to be below 500`) |
| 10 | 🟠 Medium | **Text quantity crashes the server (500).** `"quantity": "two"` → `int -= str` raises a `TypeError`. | `create_order` | `Create Order - Quantity Is Text` (`expected 500 to be below 500`) |
| 11 | 🟠 Medium | **A JSON `null` body crashes login and orders (500).** `request.json` is `None`, so `data.get(...)` raises an `AttributeError`. | `login`, `create_order` | `Login - Null JSON Body`, `Create Order - Null JSON Body` (`expected 500 to be below 500`) |
| 12 | 🟠 Medium | **A non-existent order returns `200 OK`.** `GET /orders/ORD-doesnotexist` returns `{"error": "Order not found"}` with status **200** instead of **404**. Clients that only check the status code would think the order exists. | `get_order` (`return ..., 200`) | `Get Order - Non-Existent Id` → *status is 404* (`expected 200 to equal 404`) |
| 13 | 🟠 Medium | **Debug mode leaks source code.** The app runs with `debug=True`, so every crash (bugs 9–11) returns the Werkzeug debugger HTML page with the Python traceback and lines of `app.py`, instead of a JSON error. | `app.run(debug=True)` | *error response is JSON, not an HTML page* (`expected 'text/html' to include 'application/json'`) and *no stack trace or source code is leaked* in all 4 crash tests |

### ✅ What works correctly (tests pass)
- Login with valid credentials returns a UUID token. A wrong password or missing fields → 401.
- `POST /orders` without a token or with a fake token → 401.
- Unknown `itemId` → 404 `Item not found`.
- Ordering an item whose stock is exactly 0 → 400 `Out of stock`.
- Valid orders decrease stock by the ordered quantity, and `GET /orders/<id>` returns the stored order.

---

## 🔍 How the tests detect the bugs

- **They check business rules, not only the status code.** For example, `Create Order - Valid` returns `200 OK` but still fails, because `totalPrice` does not equal `price × quantity`.
- **The expected values come from the API itself.** The menu request saves `price` and `stock` with post-response scripts. The order tests then compute the expected total and the expected stock from those values. If Auntie changes a price, the tests still work.
- **They read the state back after a change.** After each risky order, a `GET /menu` request checks the stock: it went down by the right amount, it was not changed by a rejected or invalid order, it is never negative, and it is always a whole number.
- **The negative tests expect a rejection.** Over-ordering and zero, negative, fractional, missing, text or `null` input must return `400` with an error message and **no `orderId`**. A 5xx status fails the *server does not crash* test. An HTML or traceback response fails the *no stack trace leaked* test.
- **The auth tests avoid side effects.** The *Bearer prefix* and *expired token* tests order `itemId: 999`. A correct API answers `401`. A buggy API lets the request through and answers `404 Item not found`. This proves auth was bypassed without changing any stock.
- **Auth runs on its own.** The login post-response script saves `accessToken` to the environment, and every order request uses `Bearer {{accessToken}}`. No token is pasted by hand.
- **The slow test is opt-in.** The expiry test sleeps in a pre-request script (`await bru.sleep(...)`) until the token's announced lifetime has passed. It calls `bru.runner.skipRequest()` unless `runExpiryTest=true`, so the normal run stays fast.

---

## 📁 Repo layout

```
auntie-som-noodle-api/        # Lek's Flask API (not modified)
auntie-som-noodle-api-test/   # Bruno collection "Auntie Som Lab"
  bruno.json
  collection.bru
  environments/local.bru
  01 Auth/ ... 06 Token Expiry (opt-in, slow)/
test-results/                 # output of runs on a fresh server
  cli-output.txt  report.html  junit.xml   # main run
```

> Note: in the original repo, `auntie-som-noodle-api-test` was an empty placeholder file. I replaced it with the Bruno collection folder of the same name.


---

<details>
<summary>📖 Original lab instructions</summary>


# 🍜 API Testing Lab: Auntie Som’s Noodle Stall

> **Objective:** Prove that Nephew Lek’s "it works on my machine" attitude is a recipe for disaster.

---

## 📖 The Backstory

Auntie Som runs a **legendary noodle stall**. Her Tom Yum is world-class, her customers are loyal, and her business is booming.  

Recently, her nephew **Lek** (who just finished a 2-week coding bootcamp) decided to "digitize" the business. He built a backend API to handle online orders. Auntie Som is thrilled, but the code… well, Lek says:  

> *"It’s fine lah Auntie! I tested it once with my own phone. No need for professional testing!"*

**You are the Last Line of Defense.**  
Before Auntie Som launches this app to 1,000+ hungry customers, you must audit the system, find the logic flaws, and automate the validation using **Bruno**.

---

## 🎯 Learning Objectives

By the end of this lab, you should be able to:
- 🖋️ Write **Bruno test scripts** for automated validation.
- 📦 Extract values from API responses using **Post-response scripts**.
- 🔐 Manage and use **Environment Variables**.
- 🧠 Verify **Business Rules**, not just HTTP status codes.
- 🚀 Catch regressions automatically before they hit production.

---

## 🛠️ Setup Instructions

### 1️⃣ Prepare the Project
- **Fork this repository** to your own GitHub account.
- **Clone it** locally to your machine.

### 2️⃣ Start the API Server
The backend is a lightweight Flask application.
- **Requirements:** Python 3.10+ installed.
- **Navigate** to the `auntie-som-noodle-api` folder.
- **Install dependencies:**  
  ```bash
  pip install flask
  ```
- **Run the server:**  
  ```bash
  python app.py
  ```
- The server will run at: `http://localhost:5000`

### 3️⃣ Setup Bruno
- Download and install [Bruno](https://www.usebruno.com).
- Create a **New Collection** named `Auntie Som Lab`.
- Create an **Environment** (e.g., `local`) and set a variable `baseUrl` to `http://localhost:5000`.

---

## 📑 API Reference

### 🔐 Authentication
`POST /auth/login` - Get an access token.
- **Body (JSON):**
  ```json
  {
    "username": "student",
    "password": "hungry"
  }
  ```
- **Response (200 OK):**
  ```json
  {
    "accessToken": "uuid-token-string",
    "expiresIn": 3600
  }
  ```

---

### 🍜 Menu
`GET /menu` - View available items and stock levels.
- **Response (200 OK):**
  ```json
  [
    { "id": 1, "name": "Tom Yum Noodles", "price": 50, "stock": 2 },
    { "id": 2, "name": "Fried Rice", "price": 45, "stock": 1 }
  ]
  ```

---

### 🛒 Orders
`POST /orders` - Place a new order.
- **Headers:** `Authorization: Bearer <token>`
- **Body (JSON):**
  ```json
  {
    "itemId": 1,
    "quantity": 1
  }
  ```
- **Response (200 OK):**
  ```json
  {
    "orderId": "ORD-abc123",
    "status": "created",
    "totalPrice": 45
  }
  ```

`GET /orders/<orderId>` - Retrieve order details.
- **Response (200 OK):**
  ```json
  {
    "orderId": "ORD-abc123",
    "status": "created",
    "totalPrice": 45
  }
  ```

---

## 🕵️ Your Mission: The Silent Auditor

Your task is to create a comprehensive Bruno collection that validates the entire workflow.  

⚠️ **The Catch:** Nephew Lek left several **logic bugs** in the system. Some endpoints might return `200 OK` even when the business logic is completely broken.

**Your collection must include:**
1.  **Auth Flow**: Automatically store the `accessToken` from login and use it for subsequent requests.
2.  **Order Validation**: Ensure orders can only be placed with valid auth, valid items, and available stock.
3.  **Data Integrity**: Check that the `totalPrice` calculated by the API matches your expectations.
4.  **Edge Cases**: What happens if you order more than what's in stock? What happens if you request a non-existent order?

> [!TIP]
> **Clicking "Send" is not testing.** Your tests must contain scripts (Assertions or Javascript) that **FAIL** when the system behaves incorrectly.

---

## 🧠 Rules of the Lab
- ❌ **Do NOT** modify the `app.py` backend code.
- ❌ **Do NOT** rely on manual checking.
- ✅ **Use environment variables** for the Base URL and Tokens.
- ✅ **Use Scripts/Assertions** for all validations.

---

## 📦 What to Submit

1.  **Push your Bruno collection folder** to your GitHub repository.
2.  **Update your README.md** (the one in your fork) with:
    - A summary of the bugs you discovered.
    - How your tests detect these bugs.
4.  **Send the GitHub link** to your instructor via Discord.

---
*Good luck. Auntie Som is counting on you!* 🍜🔥

</details>
