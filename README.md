# Order Management System

A full-stack order management web app built with **React**, **Node.js/Express**, and **PostgreSQL**, containerized with Docker. Supports creating orders, managing order lifecycle, real-time inventory tracking, and order cancellation with automatic stock rollback.

---

## Features

- **Order Management** - View, create, and manage orders with sortable columns (ID, quantity, amount, date)
- **Order Cancellation** - Cancel `pending` or `confirmed` orders with a confirmation dialog; automatically restores product inventory in the same DB transaction
- **Inventory Tracking** - Inventory is decremented on order creation and restored on cancellation, both within atomic transactions to prevent race conditions
- **Customer Search** - Search customers by name with a parameterized ILIKE query (SQL injection safe)
- **Status Workflow** - Orders move through `pending → confirmed → shipped → delivered`; shipped/delivered orders are locked from cancellation
- **Create Order Flow** - Select customer, product (with live price and stock preview), quantity, and shipping address; button disables during submission to prevent duplicate orders

---

## Tech Stack

| Layer | Tech |
|---|---|
| Frontend | React (CRA), vanilla CSS |
| Backend | Node.js, Express |
| Database | PostgreSQL 15 |
| Infra | Docker, Docker Compose |

---

## Architecture

```
┌─────────────────┐     HTTP      ┌──────────────────────┐     SQL      ┌──────────────┐
│  React Frontend │ ───────────▶  │  Express REST API     │ ──────────▶  │  PostgreSQL  │
│   :3000         │               │  :3001/api            │              │  :5432       │
└─────────────────┘               └──────────────────────┘              └──────────────┘
```

**API Routes:**

```
GET    /api/orders              - list all orders (JOIN with customers + products)
GET    /api/orders/:id          - single order
POST   /api/orders              - create order (transaction: insert + decrement inventory)
PATCH  /api/orders/:id/status   - update order status
PATCH  /api/orders/:id/cancel   - cancel order (transaction: update status + restore inventory)

GET    /api/customers           - list all customers
GET    /api/customers/search    - search by name (?name=)
GET    /api/customers/:id       - single customer
POST   /api/customers           - create customer

GET    /api/products            - list all products
GET    /api/products/:id        - single product
PATCH  /api/products/:id/inventory - update inventory count

GET    /api/health              - health check
```

---

## Key Implementation Details

### Transactional Order Creation
Order creation uses `BEGIN/COMMIT/ROLLBACK` with `SELECT ... FOR UPDATE` to lock the product row during the inventory check. This prevents the race condition where two concurrent requests both pass the stock check and oversell the last item.

```js
await client.query('BEGIN');
const product = await client.query('SELECT * FROM products WHERE id = $1 FOR UPDATE', [product_id]);
// check inventory...
await client.query('INSERT INTO orders ...');
await client.query('UPDATE products SET inventory_count = inventory_count - $1 ...', [quantity]);
await client.query('COMMIT');
```

### Order Cancellation with Inventory Rollback
Cancellation is also fully transactional - the status update and inventory restore happen atomically. If either fails, both roll back.

```js
// Only pending/confirmed can be cancelled
if (!['pending', 'confirmed'].includes(order.status)) {
  return res.status(400).json({ error: `Order cannot be cancelled: already ${order.status}` });
}
// Restore inventory + update status in one transaction
```

### N+1 Query Elimination
The orders list endpoint uses a single JOIN query instead of firing separate customer and product queries per row - keeps the list fast regardless of order count.

---

## Getting Started

**Prerequisites:** Docker and Docker Compose

```bash
git clone https://github.com/tomalmandal/Assignment-1.git
cd Assignment-1
docker compose up --build
```

| Service | URL |
|---|---|
| Frontend | http://localhost:3000 |
| Backend API | http://localhost:3001/api |
| Health check | http://localhost:3001/api/health |
| PostgreSQL | localhost:5432 |

The database auto-seeds with sample customers, products, and orders on first boot.

### Custom Credentials

Create a `.env` file in the project root:

```env
DB_USER=myuser
DB_PASSWORD=mypassword
DB_NAME=mydb
```

Docker Compose picks these up automatically.

---

## Docker Setup

- **Alpine images** - `node:18-alpine` and `postgres:15-alpine` keep image sizes minimal (~180MB vs ~1GB)
- **Layer cache optimization** - `package.json` is copied and `npm install` runs before copying source, so dependency installs are cached unless `package.json` changes
- **Multi-stage frontend build** - Stage 1 builds the React app, Stage 2 serves compiled static files via `serve`. No build tools or `node_modules` in the final image
- **DB health check** - Backend waits for `pg_isready` before starting, preventing crash-on-boot from early connection attempts
- **Restart policy** - All services use `restart: unless-stopped` for automatic recovery

---

## Project Structure

```
├── backend/
│   ├── src/
│   │   ├── config/db.js          # PostgreSQL pool (env-based config)
│   │   ├── routes/orders.js      # Order CRUD + cancel endpoint
│   │   ├── routes/customers.js   # Customer CRUD + search
│   │   ├── routes/products.js    # Product CRUD + inventory update
│   │   └── index.js              # Express app + error handler
│   ├── Dockerfile
│   └── package.json
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── OrderList.js      # Orders table with cancel + sort
│   │   │   ├── CreateOrder.js    # Order creation form
│   │   │   └── CustomerSearch.js # Customer search UI
│   │   ├── api/index.js          # Axios API calls
│   │   └── App.js                # Tab navigation
│   ├── Dockerfile
│   └── package.json
├── db/
│   └── init.sql                  # Schema + seed data
├── docker-compose.yml
└── BUG_REPORT.md
```

---

## Database Schema

```sql
customers   (id, name, email, phone, created_at)
products    (id, name, description, price, inventory_count, created_at)
orders      (id, customer_id, product_id, quantity, total_amount,
             status, shipping_address, created_at, updated_at)
```

Order status values: `pending | confirmed | shipped | delivered | cancelled`
