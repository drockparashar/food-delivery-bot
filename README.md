# 🍕 Food Delivery Bot

A Telegram-based food ordering bot designed for hostel environments. Students can browse a menu, select their hostel as a delivery address, add items to their order, and pay via a Cashfree payment link — all without leaving Telegram. On successful payment, the admin bot receives a full order summary automatically.

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Data Model](#data-model)
- [Order Flow](#order-flow)
- [Menu & Pricing](#menu--pricing)
- [Environment Variables](#environment-variables)
- [Getting Started](#getting-started)
- [Available Scripts](#available-scripts)
- [API Endpoints](#api-endpoints)
- [Payment Integration](#payment-integration)
- [Admin Notifications](#admin-notifications)

---

## Features

- 🏠 **Hostel selection** — users pick their delivery address (Hostel A / B / C / D) from a Telegram keyboard
- 🍔 **Inline menu** — users choose items (Pizza, Burger, Salad) via inline buttons
- 🔢 **Quantity input** — after selecting an item, the user types the desired quantity
- 🛒 **Checkout** — typing "Checkout" or pressing the Checkout button generates a Cashfree payment link
- 💳 **Online payments** — payment links are created on the fly via the Cashfree Sandbox API
- ✅ **Payment confirmation** — a webhook receives Cashfree events and notifies the customer when payment is successful
- 📋 **Admin notifications** — a separate admin bot receives a full order summary (items, quantities, total, address) after each confirmed payment
- 🗄️ **Persistent storage** — all orders and payment state are stored in MongoDB

---

## Tech Stack

| Layer | Technology |
|---|---|
| Bot framework | [Telegraf](https://telegraf.js.org/) v4 |
| Runtime | Node.js |
| Web server | [Express](https://expressjs.com/) v4 |
| Database | [MongoDB](https://www.mongodb.com/) via [Mongoose](https://mongoosejs.com/) |
| Payments | [Cashfree](https://www.cashfree.com/) Payment Links API |
| HTTP client | [Axios](https://axios-http.com/) |
| Config | [dotenv](https://github.com/motdotla/dotenv) |
| Unique IDs | [uuid](https://github.com/uuidjs/uuid) v4 |
| Signature verification | Node.js built-in `crypto` (HMAC-SHA256) |

---

## Architecture

```
Telegram User
     │
     ▼
[Telegraf Bot]  ←──────────────────────────────┐
     │                                          │
     │  store/read orders                       │
     ▼                                          │
[MongoDB]                                       │
     │                                          │
     │  generate payment link                   │
     ▼                                          │
[Cashfree API]                                  │
     │                                          │
     │  payment event (POST /cashfree-webhook)  │
     ▼                                          │
[Express Webhook Handler] ──────────────────────┘
     │
     │  order summary
     ▼
[Telegraf Admin Bot]  →  Admin Telegram Chat
```

Two separate Telegram bots run in the same process:
- **Customer bot** (`BOT_TOKEN`) — interacts with end users.
- **Admin bot** (`ADMIN_BOT_TOKEN`) — forwards confirmed order summaries to the admin chat.

---

## Project Structure

```
food-delivery-bot/
├── src/
│   ├── app.js          # Main entry point: bot logic + Express server (port 8080)
│   ├── db.js           # MongoDB connection helper (exported utility)
│   └── models/
│       └── User.js     # Mongoose schema for users & orders
├── webhook/
│   └── webhook.js      # Standalone webhook server with HMAC verification (port 3000)
├── .gitignore
├── package.json
└── README.md
```

### `src/app.js`
The heart of the application. It:
1. Connects to MongoDB.
2. Registers all Telegraf bot handlers (start, hostel selection, menu, quantity, checkout).
3. Runs an Express server on **port 8080** exposing the `/cashfree-webhook` endpoint.
4. Launches both the customer bot and the admin bot.

### `src/db.js`
An exported `connectToDB` helper that wraps `mongoose.connect`. Not used directly by `app.js` (which calls mongoose inline), but available for use in other modules.

### `src/models/User.js`
Mongoose model that persists user sessions across restarts.

### `webhook/webhook.js`
An alternative, standalone Express webhook server on **port 3000** that validates incoming Cashfree webhooks using HMAC-SHA256 signature verification. Run separately with `npm run webhook`.

---

## Data Model

```js
// User
{
  chatId:        String,   // Telegram chat ID (unique)
  address:       String,   // Selected hostel (e.g. "Hostel A")
  orders: [{
    item:        String,   // e.g. "Pizza"
    quantity:    Number,
  }],
  paymentId:     String,   // Cashfree link_id
  paymentStatus: String,   // "pending" | "confirmed" | "paid"
}
```

---

## Order Flow

```
1. /start              → User greeted, keyboard with hostel options shown
2. Select hostel       → User record upserted in MongoDB with selected address
3. Menu shown          → Inline buttons: Pizza, Burger, Salad, Checkout
4. Select item         → Item added to orders array (quantity = 0)
5. Enter quantity      → orders.$.quantity updated in MongoDB
6. Type "Checkout"     → Order summary shown + Cashfree payment link sent
7. User pays           → Cashfree POSTs to /cashfree-webhook
8. Webhook fires       → paymentStatus updated to "confirmed"
                       → Customer notified via bot.telegram.sendMessage
                       → Admin receives full order summary via adminBot
```

---

## Menu & Pricing

| Item   | Price (INR) |
|--------|-------------|
| 🍕 Pizza  | ₹200        |
| 🍔 Burger | ₹150        |
| 🥗 Salad  | ₹100        |

The total order amount is calculated as `Σ (price × quantity)` for all items with quantity > 0.

---

## Environment Variables

Create a `.env` file in the project root (it is git-ignored):

```env
# Telegram
BOT_TOKEN=<your-telegram-bot-token>
ADMIN_BOT_TOKEN=<your-admin-telegram-bot-token>
ADMIN_CHAT_ID=<admin-telegram-chat-id>

# MongoDB
MONGO_URI=<your-mongodb-connection-string>

# Cashfree
CASHFREE_CLIENT_ID=<your-cashfree-client-id>
CASHFREE_CLIENT_SECRET=<your-cashfree-client-secret>
CASHFREE_WEBHOOK_SECRET=<your-cashfree-webhook-secret>
```

| Variable | Description |
|---|---|
| `BOT_TOKEN` | Token for the customer-facing Telegram bot (from [@BotFather](https://t.me/BotFather)) |
| `ADMIN_BOT_TOKEN` | Token for the admin notification bot (can be a second bot or the same bot token) |
| `ADMIN_CHAT_ID` | Telegram chat ID of the admin who receives order notifications |
| `MONGO_URI` | MongoDB connection URI (e.g. `mongodb+srv://...` for Atlas) |
| `CASHFREE_CLIENT_ID` | App ID from the [Cashfree Dashboard](https://merchant.cashfree.com/) |
| `CASHFREE_CLIENT_SECRET` | Secret key from the Cashfree Dashboard |
| `CASHFREE_WEBHOOK_SECRET` | Webhook secret used to verify HMAC signatures in `webhook/webhook.js` |

---

## Getting Started

### Prerequisites

- Node.js ≥ 18
- A running MongoDB instance (local or [MongoDB Atlas](https://www.mongodb.com/atlas))
- A Telegram bot token from [@BotFather](https://t.me/BotFather)
- A [Cashfree](https://www.cashfree.com/) merchant account (sandbox is fine for testing)

### Installation

```bash
# 1. Clone the repository
git clone https://github.com/drockparashar/food-delivery-bot.git
cd food-delivery-bot

# 2. Install dependencies
npm install

# 3. Create and fill in the environment file
cp .env.example .env   # or create .env manually — see above
```

### Running the Bot

```bash
# Start the main bot + webhook server on port 8080
npm start
```

To also run the standalone HMAC-verified webhook server on port 3000:

```bash
npm run webhook
```

> **Note:** Both servers can run simultaneously in separate terminal windows or processes. In production, consider a process manager such as [PM2](https://pm2.keymetrics.io/).

---

## Available Scripts

| Command | Description |
|---|---|
| `npm start` | Start the main bot and Express server (`src/app.js`, port 8080) |
| `npm run webhook` | Start the standalone webhook server (`webhook/webhook.js`, port 3000) |

---

## API Endpoints

### `POST /cashfree-webhook` — (main server, port 8080)

Receives payment event notifications from Cashfree.

**Expected payload structure (Cashfree format):**
```json
{
  "data": {
    "link_id": "<cashfree-link-id>",
    "link_status": "PAID",
    "order": {
      "transaction_status": "SUCCESS"
    }
  }
}
```

**Behaviour:**
- If `transaction_status === 'SUCCESS'`, finds the matching user by `paymentId`, updates `paymentStatus` to `"confirmed"`, sends a Telegram message to the customer, and forwards the order summary to the admin.
- Returns `200 OK` on success, `500` on error.

---

### `POST /webhook` — (standalone server, port 3000)

Receives payment events with HMAC-SHA256 signature verification.

**Headers required:**
```
x-cashfree-signature: <hmac-sha256-signature>
```

**Behaviour:**
- Verifies the request signature using `CASHFREE_WEBHOOK_SECRET`.
- On valid signature and `order_status === 'PAID'`, updates `paymentStatus` to `"paid"` for the matching user.
- Returns `400 Bad Request` if the signature does not match.

---

## Payment Integration

The bot uses the **Cashfree Payment Links API** (sandbox environment). When a user checks out:

1. A `POST` request is sent to `https://sandbox.cashfree.com/pg/links` with:
   - A unique `link_id` (UUID v4)
   - The total order amount in INR
   - The customer's phone number
   - The order purpose string
2. The returned `link_url` is sent to the user in Telegram.
3. After the user pays, Cashfree calls the configured webhook URL.

> To use in production, replace the sandbox URL with `https://api.cashfree.com/pg/links` and update your Cashfree dashboard webhook URL to point to your public server.

---

## Admin Notifications

After a payment is confirmed, the admin bot sends a message to `ADMIN_CHAT_ID` with the full order summary, for example:

```
New Order Received:

Address: Hostel B
Items:
- 2 Pizza(s)
- 1 Burger(s)

Total Amount: ₹550
```

To find your admin chat ID, start the admin bot and check the console — it logs the chat ID on the `/start` command.
