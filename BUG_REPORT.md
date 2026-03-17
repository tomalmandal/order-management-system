# Bug Report - Order Management System

---

## Bug 1: SQL Injection in Customer Search
**Where:** `backend/src/routes/customers.js` - `/search` route
**Severity:** Critical
**Found while:** Reading the customer search route and noticed the query was built using string concatenation instead of parameters.

**What's wrong:**
The search query pastes user input directly into the SQL string:
```js
const query = "SELECT * FROM customers WHERE name ILIKE '%" + name + "%'";
```
If someone types `' OR '1'='1` in the search box, the query breaks and returns all rows. Worse, something like `'; DROP TABLE customers; --` could wipe the table. Every other query in this codebase uses `$1` placeholders correctly - this one was missed.

**Expected behavior:** Search should only return customers matching the name, nothing else.

**Fix:**
```js
const result = await pool.query(
  "SELECT * FROM customers WHERE name ILIKE $1",
  [`%${name}%`]
);
```

---

## Bug 2: N+1 Query Problem in GET /orders
**Where:** `backend/src/routes/orders.js` - `GET /` route
**Severity:** Critical
**Found while:** Reading the orders route and noticed a loop with DB calls inside it.

**What's wrong:**
The route first fetches all orders, then for each order it fires 2 more queries (one for customer, one for product) inside a `for` loop:
```js
for (const order of orders) {
  const customerResult = await pool.query('SELECT name, email FROM customers WHERE id = $1', [order.customer_id]);
  const productResult = await pool.query('SELECT name, price FROM products WHERE id = $1', [order.product_id]);
}
```
With 10 orders that's 21 DB queries. With 100 orders it's 201. The funny thing is the single-order route `GET /:id` already does this correctly with a JOIN - so this looks like an oversight when writing the list endpoint.

**Expected behavior:** Fetching all orders should take 1 DB query, not 2N+1.

**Fix:** Use a JOIN like the single order endpoint already does:
```js
const result = await pool.query(`
  SELECT o.*, c.name as customer_name, c.email as customer_email,
         p.name as product_name, p.price as product_price
  FROM orders o
  JOIN customers c ON o.customer_id = c.id
  JOIN products p ON o.product_id = p.id
  ORDER BY o.created_at DESC
`);
```

---

## Bug 3: No Database Transaction on Order Creation
**Where:** `backend/src/routes/orders.js` - `POST /` route
**Severity:** Critical
**Found while:** Tracing the order creation flow step by step.

**What's wrong:**
Creating an order has two steps: insert the order row, then decrement inventory. These run as two separate queries with nothing connecting them:
```js
// Step 1
const orderResult = await pool.query(`INSERT INTO orders ...`);

// Step 2 - runs independently, no rollback if this fails
await pool.query('UPDATE products SET inventory_count = inventory_count - $1 WHERE id = $2', ...);
```
If the server crashes or the DB drops between these two lines, you end up with an order in the database but full inventory - meaning the stock was never reduced. There's also a race condition: if two users order the last item at the same time, both can pass the inventory check before either one decrements it, resulting in negative stock.

**Expected behavior:** Both the order insert and inventory decrement should succeed or fail together.

**Fix:** Wrap both queries in a transaction with `BEGIN / COMMIT / ROLLBACK` and use `SELECT ... FOR UPDATE` to lock the product row during the check:
```js
const client = await pool.connect();
try {
  await client.query('BEGIN');
  const productResult = await client.query(
    'SELECT * FROM products WHERE id = $1 FOR UPDATE', [product_id]
  );
  // check inventory...
  await client.query(`INSERT INTO orders ...`);
  await client.query('UPDATE products SET inventory_count = inventory_count - $1 WHERE id = $2', ...);
  await client.query('COMMIT');
} catch (err) {
  await client.query('ROLLBACK');
  res.status(500).json({ error: 'Failed to create order' });
} finally {
  client.release();
}
```

---

## Bug 4: Error Handler Returns 200 OK for Every Error
**Where:** `backend/src/index.js` - global error handler
**Severity:** Critical
**Found while:** Reading the bottom of index.js.

**What's wrong:**
The global error handler sends a success response for every error:
```js
app.use((err, req, res, next) => {
  console.log('Something happened');
  res.status(200).json({ success: true });
});
```
Any unhandled exception in the app will tell the client "everything is fine" when it isn't. The frontend will think operations worked when they silently failed. The `console.log('Something happened')` also throws away the actual error, making debugging nearly impossible.

**Expected behavior:** Errors should return an appropriate status code (500) with an error message, and the full error should be logged.

**Fix:**
```js
app.use((err, req, res, next) => {
  console.error('Unhandled error:', err);
  res.status(500).json({ error: 'Internal server error' });
});
```

---

## Bug 5: DB Credentials Hardcoded in Source Code
**Where:** `backend/src/config/db.js`
**Severity:** High
**Found while:** Opening the DB config file.

**What's wrong:**
Username and password are written directly in the code:
```js
const pool = new Pool({
  user: 'admin',
  password: 'admin123',
  ...
});
```
Anyone who can read this file - or the git history - has the DB password. Even if the repo is private today, rotating credentials later means changing code and redeploying, which is messy.

**Expected behavior:** Credentials should come from environment variables so they can be changed without touching code.

**Fix:**
```js
const pool = new Pool({
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  host: process.env.DB_HOST || 'db',
  port: parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME,
});
```
And pass these in via `docker-compose.yml` environment section.

---

## Bug 6: Missing useEffect Dependency Breaks Product Preview
**Where:** `frontend/src/components/CreateOrder.js`
**Severity:** Medium
**Found while:** Looking at the useEffect hooks in CreateOrder.

**What's wrong:**
There's a `useEffect` meant to update the selected product details whenever the user picks a product. But `selectedProduct` is missing from the dependency array:
```js
useEffect(() => {
  if (selectedProduct) {
    const product = products.find(p => p.id === parseInt(selectedProduct));
    setSelectedProductData(product);
  }
}, [products]); // selectedProduct is missing here
```
This effect only runs when `products` loads (once on mount). Changing the dropdown after that does nothing - the preview panel never updates.

**Expected behavior:** Selecting a different product should immediately update the price and stock preview.

**Fix:**
```js
}, [products, selectedProduct]);
```

---

## Bug 7: No Input Validation on Order Creation
**Where:** `backend/src/routes/orders.js` - `POST /` route
**Severity:** High
**Found while:** Looking at what the POST /orders handler does with req.body.

**What's wrong:**
The handler uses `customer_id`, `product_id`, `quantity`, and `shipping_address` directly from the request body without checking if they exist or make sense. Someone could send `quantity: -5` which would pass the inventory check and actually *increase* stock instead of decreasing it. Missing fields cause raw DB errors to bubble up.

**Expected behavior:** The API should reject bad input with a clear error before touching the database.

**Fix:**
```js
if (!customer_id || !product_id || !shipping_address) {
  return res.status(400).json({ error: 'Missing required fields' });
}
if (!Number.isInteger(quantity) || quantity < 1) {
  return res.status(400).json({ error: 'Quantity must be a positive integer' });
}
```

---

## Bug 8: No DB Health Check in docker-compose
**Where:** `docker-compose.yml`
**Severity:** Medium
**Found while:** Reading the docker-compose file and checking service dependencies.

**What's wrong:**
The backend `depends_on: db` only waits for the container to start, not for Postgres to actually be ready to accept connections. Postgres takes a few seconds to initialize, so the backend often crashes on first boot with a connection error. There's also no `restart` policy, so it just stays dead.

**Expected behavior:** The backend should only start once the DB is actually ready, and should restart automatically if it crashes.

**Fix:**
```yaml
db:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U admin -d orderdb"]
    interval: 5s
    timeout: 5s
    retries: 5

backend:
  depends_on:
    db:
      condition: service_healthy
  restart: on-failure
```

---

## Bug 9: Place Order Button Stays Active During API Call
**Where:** `frontend/src/components/CreateOrder.js` - `handleSubmit` function
**Severity:** Medium
**Found while:** Looking at the form submission handler.

**What's wrong:**
The "Place Order" button has no loading or disabled state while the request is being processed. If a user clicks it more than once (slow connection, double-click), multiple POST requests go out and multiple orders get created - each one also decrementing inventory.

**Expected behavior:** The button should be disabled from the moment the request starts until a response comes back.

**Fix:**
```js
const [loading, setLoading] = useState(false);

const handleSubmit = async () => {
  setLoading(true);
  const result = await createOrder({...});
  setLoading(false);
  ...
};

<button disabled={loading} onClick={handleSubmit}>
  {loading ? 'Placing...' : 'Place Order'}
</button>
```

---

## Summary

| # | Issue | File | Severity |
|---|-------|------|----------|
| 1 | SQL injection in customer search | `routes/customers.js` | Critical |
| 2 | N+1 queries in GET /orders | `routes/orders.js` | Critical |
| 3 | No DB transaction on order creation | `routes/orders.js` | Critical |
| 4 | Error handler returns 200 OK for all errors | `src/index.js` | Critical |
| 5 | Hardcoded DB credentials | `config/db.js` | High |
| 6 | Missing useEffect dependency | `CreateOrder.js` | Medium |
| 7 | No input validation on POST /orders | `routes/orders.js` | High |
| 8 | No DB healthcheck in docker-compose | `docker-compose.yml` | Medium |
| 9 | Button not disabled during API call | `CreateOrder.js` | Medium |
