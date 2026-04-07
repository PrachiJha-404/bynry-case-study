# Inventory Management System for B2B SaaS
## SOLUTIONS

---

## Part 1: Code Review & Debugging (30 minutes)

```python
@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json
    
    # Create new product
    product = Product(
        name=data['name'],
        sku=data['sku'], 
        price=data['price'],
        warehouse_id=data['warehouse_id']
    )
    
    db.session.add(product)
    db.session.commit()     
    # Update inventory count
    inventory = Inventory(
        product_id=product.id,
        warehouse_id=data['warehouse_id'],
        quantity=data['initial_quantity']
    )     
    db.session.add(inventory) 
    db.session.commit()
    
    return {"message": "Product created", "product_id": product.id}
```

### Your Tasks:

**1 and 2. Identify Issues:** List all problems you see with this code (technical and business logic). **Explain Impact:** For each issue, explain what could go wrong in production

1. **No input validation** - The code does `data['name']` directly. If the field is missing, you get a raw KeyError (500 error, no useful message to the client). Optional fields like price formatting or warehouse existence aren't checked either. SQL injection however isn't a risk here specifically because we're using an ORM with parameterized queries.
2. **SKU not enforced as unique** - The requirements state that SKU must be unique, however no check or constraints for this are enforced in the code.
3. **Split transactions** - Two `db.session.commit()` calls. If the Inventory insert fails, you have an orphaned Product row with no inventory. Breaks data integrity.
4. **No error handling** - No try/except anywhere. DB errors, constraint violations, and missing fields all crash with a 500 and expose stack traces.
5. **No HTTP status codes** - A successful creation should return 201 Created, not an implicit 200 OK.
6. **Price has no validation** - Negative prices, non-numeric strings, or missing values will either crash or silently store garbage.
7. **initial_quantity not validated** - Could be negative or missing.
8. **Warehouse existence not verified** - The code assumes warehouse_id is valid. A bad ID either crashes or creates an inventory record pointing to a non-existent warehouse.

**Provide Fixes:** Write the corrected version with explanations

```python
from flask import request, jsonify
from sqlalchemy.exc import IntegrityError
from decimal import Decimal, InvalidOperation

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json

    # Input validation
    required_fields = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']
    missing = [f for f in required_fields if f not in data]
    if missing:
        return jsonify({"error": f"Missing required fields: {missing}"}), 400

    try:
        price = Decimal(str(data['price']))
        if price < 0:
            raise ValueError()
    except (InvalidOperation, ValueError):
        return jsonify({"error": "Price must be a non-negative number"}), 400

    quantity = data['initial_quantity']
    if not isinstance(quantity, int) or quantity < 0:
        return jsonify({"error": "initial_quantity must be a non-negative integer"}), 400

    warehouse = Warehouse.query.get(data['warehouse_id'])
    if not warehouse:
        return jsonify({"error": "Warehouse not found"}), 404

    try:
        # Create product and inventory in a SINGLE transaction.    
        product = Product(
            name=data['name'],
            sku=data['sku'],       # Add unique constraint in DB column itself for sku
            price=price,
            warehouse_id=data['warehouse_id']
        )
        db.session.add(product)
        db.session.flush()  

        inventory = Inventory(
            product_id=product.id,
            warehouse_id=data['warehouse_id'],
            quantity=quantity
        )
        db.session.add(inventory)
        db.session.commit()  # One single commit to make it atomic

    except IntegrityError:
        db.session.rollback()   
        return jsonify({"error": f"SKU '{data['sku']}' already exists"}), 409

    except Exception as e:
        db.session.rollback()   
        return jsonify({"error": "An unexpected error occurred"}), 500

    return jsonify({"message": "Product created", "product_id": product.id}), 201
```

---

## Part 2: Database Design (25 minutes)

### Your Tasks:

**Design Schema:** Create tables with columns, data types, and relationships

**DDL for the design schema**

```sql
-- Core company entity
CREATE TABLE companies (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- Warehouses belong to a company
CREATE TABLE warehouses (
    id          SERIAL PRIMARY KEY,
    company_id  INT NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    name        VARCHAR(255) NOT NULL,
    location    TEXT,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- Products belong to a company; SKU unique within a company
CREATE TABLE products (
    id          SERIAL PRIMARY KEY,
    company_id  INT NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    name        VARCHAR(255)    NOT NULL,
    sku         VARCHAR(100)    NOT NULL,
    price       NUMERIC(12, 2)  NOT NULL CHECK (price >= 0),
    product_type VARCHAR(50)  NOT NULL DEFAULT 'standard', -- 'standard' | 'bundle'
    low_stock_threshold INT NOT NULL DEFAULT 50,  -- per-product threshold, not hardcoded globally
    created_at  TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    UNIQUE (company_id, sku)    -- SKU unique per company, not globally (two companies can use "WID-001")
);

-- For bundle products: which products make up the bundle
CREATE TABLE bundle_components (
    bundle_product_id   INT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    component_product_id INT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    quantity            INT NOT NULL CHECK (quantity > 0),
    PRIMARY KEY (bundle_product_id, component_product_id)
);

-- Current stock level: one row per (product, warehouse) pair
CREATE TABLE inventory (
    id              SERIAL PRIMARY KEY,
    product_id      INT  NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    warehouse_id    INT  NOT NULL REFERENCES warehouses(id) ON DELETE CASCADE,
    quantity        INT  NOT NULL DEFAULT 0 CHECK (quantity >= 0),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (product_id, warehouse_id)
);

-- Audit log of every inventory change
CREATE TABLE inventory_events (
    id              SERIAL PRIMARY KEY,
    product_id      INT         NOT NULL REFERENCES products(id),
    warehouse_id    INT         NOT NULL REFERENCES warehouses(id),
    delta           INT         NOT NULL,  -- positive = stock added, negative = stock removed
    event_type      VARCHAR(50) NOT NULL,  -- 'sale', 'restock', 'adjustment', 'transfer'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Suppliers are independent entities 
CREATE TABLE suppliers (
    id              SERIAL PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    contact_email   VARCHAR(255),
    phone           VARCHAR(50),
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- a product can have multiple suppliers; a supplier supplies many products
CREATE TABLE product_suppliers (
    product_id      INT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    supplier_id     INT NOT NULL REFERENCES suppliers(id) ON DELETE CASCADE,
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,  -- flag the preferred reorder supplier
    lead_time_days  INT,                           
    PRIMARY KEY (product_id, supplier_id)
);

-- Indexes for the queries that will run constantly
CREATE INDEX idx_inventory_warehouse ON inventory(warehouse_id);
CREATE INDEX idx_inventory_product   ON inventory(product_id);
CREATE INDEX idx_events_product_time ON inventory_events(product_id, created_at DESC);
CREATE INDEX idx_products_company    ON products(company_id);
```

**Identify Gaps:** List questions you'd ask the product team about missing requirements

These are the questions I would ask about the missing requirements:

1. Is SKU unique globally or per-company? (I assumed per-company above)
2. How is "recent sales activity" defined exactly? Is there a specific number of days? Is that configurable?
3. Who sets the low-stock threshold? Per product? Per product-type category? Company-wide default?
4. What triggers an inventory event? Is there an order management system, or does stock change manually?
5. Can stock go negative? (e.g., backorders). I added `CHECK (quantity >= 0)` but this might be wrong.
6. Are suppliers global or per-company? Can two companies share the same supplier record, or does each company maintain its own supplier list?
7. What does "bundle" mean for inventory? Does selling a bundle deduct from component stock individually?
8. Multi-currency? Price is stored in what currency?
9. Soft deletes? Should deleted products/warehouses be archived rather than hard-deleted?
10. Are transfers between warehouses a first-class concept? (e.g., move 50 units from Warehouse A to B.)

**Explain Decisions:** Justify your design choices (indexes, constraints, etc.)

#### Constraints

**`UNIQUE (company_id, sku)` instead of `UNIQUE (sku)`**
SKU only needs to be unique within a company's catalog. A global unique constraint would mean two completely separate businesses can't both use "WID-001", which is unnecessarily restrictive and would cause artificial conflicts at scale.

**`CHECK (price >= 0)`**
Prevents garbage data at the DB level, not just the application level. Application validation can be bypassed (direct DB access, migrations, scripts) — the constraint cannot.

**`CHECK (quantity >= 0)` on inventory**
Prevents negative stock unless we explicitly decide to allow backorders. Better to be strict now and relax it later than to discover negative quantities corrupting reports.

**`CHECK (quantity > 0)` on bundle_components**
A bundle containing zero of something is meaningless. Enforced at the DB level so no application bug can create it.

**`ON DELETE CASCADE` on warehouses, products, inventory**
If a company is deleted, all its data goes with it cleanly. If a product is deleted, its inventory rows don't become orphans pointing at a non-existent product. This is safer than leaving dangling foreign keys, though it assumes hard deletes — worth discussing with the team (gap #9 in your list).

**`NUMERIC(12, 2)` for price instead of `FLOAT`**
`FLOAT` is a binary approximation — 0.1 + 0.2 in floating point is 0.30000000000000004. For money, this causes real rounding errors at scale. `NUMERIC` stores exact decimal values.

**`TIMESTAMPTZ` instead of `TIMESTAMP`**
Stores timezone-aware timestamps. If your servers or users are ever in different timezones (almost certain for a B2B SaaS), naive timestamps become ambiguous and cause ordering bugs. Always use `TIMESTAMPTZ`.

#### Table Design Decisions

**Separate `inventory` and `inventory_events` tables**
`inventory` holds the current quantity - one row per (product, warehouse). It's what you read for stock checks. `inventory_events` is an append-only audit log of every change. You never update events, only insert them. This separation means:
- Fast reads from `inventory` (single row lookup)
- Full history available from `inventory_events` for analytics, debugging, and the days_until_stockout calculation
- If you only had events, computing current stock would require summing all deltas every time - slow and fragile

**`delta` as signed integer in `inventory_events`**
A sale is -5, a restock is +20. This means you can always reconstruct current stock by `SUM(delta)` and verify it matches the inventory table. Clean and auditable.

**`bundle_components` as a self-referencing junction table**
Bundle products reference other products in the same table. This handles arbitrary bundle depth without schema changes. The tradeoff is you need application logic to prevent circular references (a bundle containing itself), since SQL constraints can't easily catch that.

**`product_suppliers` junction with `is_primary` and `lead_time_days`**
Many-to-many because one product can have multiple suppliers and one supplier supplies many products. `is_primary` lets the alert system know which supplier to surface for reordering without requiring a separate query. `lead_time_days` is stored per relationship, not per supplier globally, because the same supplier might deliver Product A in 3 days and Product B in 14 days.

**`suppliers` as a global table, not per-company**
A supplier like "Acme Corp" is one real-world entity. If two companies both buy from Acme, they should reference the same supplier row. This avoids duplicate supplier records and allows future features like "show all companies buying from this supplier." This is an assumption - flagged as gap #6.

#### Index Decisions

**`idx_inventory_warehouse` and `idx_inventory_product`**
The most common inventory queries are "what's in this warehouse?" and "where is this product stocked?" Without indexes, these scan the entire inventory table. With them, it's a direct lookup.

**`idx_events_product_time` - composite, descending on `created_at`**
The low-stock alert query filters events by `product_id` AND `created_at >= cutoff`. The composite index serves both filters in one. Descending on time matches the query pattern of "most recent first." A single-column index on just `product_id` would still require scanning all time periods.

**`idx_products_company`**
Every meaningful product query starts with "for this company." Without this, every product lookup scans the whole products table across all companies - a multi-tenancy performance disaster at scale.

#### ER Diagram

*(See original document)*

---

## Part 3: API Implementation (35 minutes)

Implement an endpoint that returns low-stock alerts for a company.

**Business Rules (discovered through previous questions):**
- Low stock threshold varies by product type
- Only alert for products with recent sales activity
- Must handle multiple warehouses per company
- Include supplier information for reordering

> So how would I go about the logic? Have a minimum threshold, not hardcoded, maybe interfaces and for now we keep it 50 items? Or instead we could do a demand vs supply at any given moment based on previous 10-20 days.

**Endpoint Specification:**

```
GET /api/companies/{company_id}/alerts/low-stock
```

**Expected Response Format:**

```json
{
  "alerts": [
    {
      "product_id": 123,
      "product_name": "Widget A",
      "sku": "WID-001",
      "warehouse_id": 456,
      "warehouse_name": "Main Warehouse",
      "current_stock": 5,
      "threshold": 20,
      "days_until_stockout": 12,
      "supplier": {
        "id": 789,
        "name": "Supplier Corp",
        "contact_email": "orders@supplier.com"
      }
    }
  ],
  "total_alerts": 1
}
```

### Your Tasks:

**Write Implementation:** Use any language/framework (Python/Flask, Node.js/Express, etc.)

```go
package main

import (
	"database/sql"
	"encoding/json"
	"log"
	"math"
	"net/http"
	"strconv"
	"time"

	"github.com/lib/pq" // postgres driver
)

// ── Constants ────────────────────────────────────────────────────────────────

const (
	// How far back we look for sales activity.
	// A product with zero sales in this window is excluded — it's slow-moving,
	// not low-stock. Ask the product team if this should be configurable per company.
	recentActivityDays = 30

	// Minimum sales needed in the window to be considered "active".
	// Prevents a product sold once 29 days ago from generating urgent alerts.
	minUnitsToBeActive = 1
)

// ── Response types ────────────────────────────────────────────────────────────
// These match the spec's JSON shape exactly. Pointer fields (*int, *Supplier)
// go null in JSON when nil — e.g. a product with no supplier, or one
// where days_until_stockout can't be calculated (zero avg daily sales).

type Supplier struct {
	ID           int    `json:"id"`
	Name         string `json:"name"`
	ContactEmail string `json:"contact_email"`
}

type Alert struct {
	ProductID         int       `json:"product_id"`
	ProductName       string    `json:"product_name"`
	SKU               string    `json:"sku"`
	WarehouseID       int       `json:"warehouse_id"`
	WarehouseName     string    `json:"warehouse_name"`
	CurrentStock      int       `json:"current_stock"`
	Threshold         int       `json:"threshold"`
	DaysUntilStockout *int      `json:"days_until_stockout"` // nil → null in JSON
	Supplier          *Supplier `json:"supplier"`            // nil → null in JSON
}

type AlertsResponse struct {
	Alerts      []Alert `json:"alerts"`
	TotalAlerts int     `json:"total_alerts"`
}

// ── Handler ───────────────────────────────────────────────────────────────────

// LowStockHandler holds the DB reference. In Go this is the idiomatic way to
// share dependencies with handlers — no global state, easy to test by injecting
// a mock DB.
type LowStockHandler struct {
	db *sql.DB
}

func (h *LowStockHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// Only allow GET
	if r.Method != http.MethodGet {
		writeError(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	// Extract company_id from the URL path.
	// In production you'd use a router (chi, gorilla/mux) that does this for you.
	// Here we parse it manually from the last path segment.
	companyID, err := parseCompanyID(r.URL.Path)
	if err != nil {
		writeError(w, "invalid company_id: must be a positive integer", http.StatusBadRequest)
		return
	}

	alerts, err := h.fetchAlerts(r.Context(), companyID)
	if err != nil {
		// Check for "company not found" specifically so we can return 404.
		// All other DB errors become 500.
		if err == errCompanyNotFound {
			writeError(w, "company not found", http.StatusNotFound)
			return
		}
		log.Printf("fetchAlerts error for company %d: %v", companyID, err)
		writeError(w, "internal server error", http.StatusInternalServerError)
		return
	}

	// Always return an array, never null — easier for clients to handle.
	if alerts == nil {
		alerts = []Alert{}
	}

	writeJSON(w, AlertsResponse{
		Alerts:      alerts,
		TotalAlerts: len(alerts),
	}, http.StatusOK)
}

// ── Core query ────────────────────────────────────────────────────────────────

var errCompanyNotFound = fmt.Errorf("company not found")

func (h *LowStockHandler) fetchAlerts(ctx context.Context, companyID int) ([]Alert, error) {
	// First verify the company exists. Without this, a bad company_id just
	// returns an empty alert list, which looks like success — misleading.
	var exists bool
	err := h.db.QueryRowContext(ctx,
		`SELECT EXISTS(SELECT 1 FROM companies WHERE id = $1)`, companyID,
	).Scan(&exists)
	if err != nil {
		return nil, err
	}
	if !exists {
		return nil, errCompanyNotFound
	}

	cutoff := time.Now().UTC().AddDate(0, 0, -recentActivityDays)

	// One query does all the heavy lifting with two CTEs:
	//
	// recent_sales:
	//   Sums all outbound (sale) events per product/warehouse in the activity
	//   window. Products with no sales in the window are excluded because
	//   the main query INNER JOINs on this CTE — intentional, not a bug.
	//   avg_daily_sales is the key metric for days_until_stockout.
	//
	// preferred_supplier:
	//   Picks one supplier per product. is_primary = TRUE wins; ties broken
	//   alphabetically. LEFT JOIN in the main query so missing suppliers → null.
	//
	// Main query:
	//   Filters to: this company's products, in this company's warehouses,
	//   where current stock is below the per-product threshold, and where
	//   the product had recent sales (via INNER JOIN on recent_sales).
	//   Results ordered most urgent first (fewest days until stockout).
	query := `
	WITH recent_sales AS (
	    SELECT
	        ie.product_id,
	        ie.warehouse_id,
	        SUM(ABS(ie.delta))::float                              AS units_sold,
	        SUM(ABS(ie.delta))::float / $1::float                 AS avg_daily_sales
	    FROM inventory_events ie
	    WHERE ie.event_type = 'sale'
	      AND ie.created_at >= $2
	    GROUP BY ie.product_id, ie.warehouse_id
	    HAVING SUM(ABS(ie.delta)) >= $3
	),
	preferred_supplier AS (
	    SELECT DISTINCT ON (ps.product_id)
	        ps.product_id,
	        s.id            AS supplier_id,
	        s.name          AS supplier_name,
	        s.contact_email AS supplier_email
	    FROM product_suppliers ps
	    JOIN suppliers s ON s.id = ps.supplier_id
	    ORDER BY ps.product_id, ps.is_primary DESC, s.name ASC
	)
	SELECT
	    p.id                                                        AS product_id,
	    p.name                                                      AS product_name,
	    p.sku,
	    w.id                                                        AS warehouse_id,
	    w.name                                                      AS warehouse_name,
	    i.quantity                                                  AS current_stock,
	    p.low_stock_threshold                                       AS threshold,
	    CASE
	        WHEN rs.avg_daily_sales > 0
	        THEN FLOOR(i.quantity::float / rs.avg_daily_sales)::int
	        ELSE NULL
	    END                                                         AS days_until_stockout,
	    ps.supplier_id,
	    ps.supplier_name,
	    ps.supplier_email
	FROM products p
	JOIN inventory i           ON i.product_id    = p.id
	JOIN warehouses w          ON w.id            = i.warehouse_id
	JOIN recent_sales rs       ON rs.product_id   = p.id
	                          AND rs.warehouse_id = i.warehouse_id
	LEFT JOIN preferred_supplier ps ON ps.product_id = p.id
	WHERE p.company_id  = $4
	  AND w.company_id  = $4
	  AND i.quantity    < p.low_stock_threshold
	ORDER BY days_until_stockout ASC NULLS LAST
	`

	rows, err := h.db.QueryContext(ctx, query,
		float64(recentActivityDays), // $1 — divisor for avg_daily_sales
		cutoff,                      // $2 — activity window start
		minUnitsToBeActive,          // $3 — HAVING threshold
		companyID,                   // $4 — tenant filter
	)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var alerts []Alert

	for rows.Next() {
		var (
			a                 Alert
			daysUntilStockout sql.NullInt64  // NULL when avg_daily_sales = 0
			supplierID        sql.NullInt64
			supplierName      sql.NullString
			supplierEmail     sql.NullString
		)

		err := rows.Scan(
			&a.ProductID,
			&a.ProductName,
			&a.SKU,
			&a.WarehouseID,
			&a.WarehouseName,
			&a.CurrentStock,
			&a.Threshold,
			&daysUntilStockout,
			&supplierID,
			&supplierName,
			&supplierEmail,
		)
		if err != nil {
			return nil, err
		}

		// Convert sql.Null* types to Go pointers.
		// A nil pointer serialises as JSON null — correct for the spec.
		if daysUntilStockout.Valid {
			d := int(daysUntilStockout.Int64)
			a.DaysUntilStockout = &d
		}

		if supplierID.Valid {
			a.Supplier = &Supplier{
				ID:           int(supplierID.Int64),
				Name:         supplierName.String,
				ContactEmail: supplierEmail.String,
			}
		}

		alerts = append(alerts, a)
	}

	// rows.Err() catches errors that happened mid-iteration —
	// rows.Next() swallows them and just returns false.
	if err := rows.Err(); err != nil {
		return nil, err
	}

	return alerts, nil
}

// ── Helpers ───────────────────────────────────────────────────────────────────

func parseCompanyID(path string) (int, error) {
	// Path: /api/companies/{company_id}/alerts/low-stock
	// We take the segment after "companies/" and before "/alerts"
	// A proper router (chi, mux) would handle this — keeping it simple here.
	segments := strings.Split(strings.Trim(path, "/"), "/")
	for i, seg := range segments {
		if seg == "companies" && i+1 < len(segments) {
			id, err := strconv.Atoi(segments[i+1])
			if err != nil || id <= 0 {
				return 0, fmt.Errorf("invalid id")
			}
			return id, nil
		}
	}
	return 0, fmt.Errorf("not found in path")
}

func writeJSON(w http.ResponseWriter, v any, status int) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v)
}

func writeError(w http.ResponseWriter, msg string, status int) {
	writeJSON(w, map[string]string{"error": msg}, status)
}

// ── Main ──────────────────────────────────────────────────────────────────────

func main() {
	db, err := sql.Open("postgres", "postgres://user:pass@localhost/stockflow?sslmode=disable")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	// Connection pool settings 
	db.SetMaxOpenConns(25)
	db.SetMaxIdleConns(10)
	db.SetConnMaxLifetime(5 * time.Minute)

	handler := &LowStockHandler{db: db}
	http.Handle("/api/companies/", handler)

	log.Println("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Handle Edge Cases:** Consider what could go wrong

**Input & routing**
- Non-integer or negative `company_id` → 400
- Non-existent `company_id` → 404 (not a silent empty list)
- Wrong HTTP method → 405

**Data nullability**
- Product has no supplier → `supplier` is null, not an error
- `avg_daily_sales` is zero → `days_until_stockout` is null, not a divide-by-zero crash
- No alerts exist for a valid company → returns `[]` not null (easier for clients to handle)

**Tenancy / security**
- `AND w.company_id = $4` double-checks warehouses belong to the requesting company — prevents cross-tenant data leaks even if a FK association is wrong

**Sales activity filtering**
- Products with zero sales in the last 30 days are excluded entirely (INNER JOIN on `recent_sales`) — slow-moving stock isn't a reorder emergency
- `HAVING SUM(...) >= minUnitsToBeActive` filters out products sold just once and never again

**Supplier selection**
- Multiple suppliers → picks `is_primary = TRUE`, falls back to alphabetical — never returns multiple supplier rows per product

**DB iteration**
- `rows.Err()` checked after the loop — catches mid-scan errors that `rows.Next()` silently swallows

**Connection safety**
- `defer rows.Close()` — connection returns to pool even if the handler returns early
- Pool limits (`SetMaxOpenConns`) — prevents connection exhaustion under load

**Ordering**
- Results sorted by `days_until_stockout ASC NULLS LAST` — most urgent alerts surface first, nulls don't bubble to the top

**Explain Approach:** Add comments explaining your logic

#### Visualized flow

*(See original document)*

---

### Documented Assumptions

1. `inventory_events.event_type = 'sale'` represents outbound stock. `restock`, `adjustment`, and `transfer` are excluded from consumption calculations.
2. "Recent sales activity" means the last 30 days (constant `recentActivityDays`). This should be asked from the product team — it might need to be per-company or configurable.
3. A product needs at least 1 unit sold in the window (`minUnitsToBeActive`) to appear in alerts. A product sold once then never again shouldn't trigger urgent reorder alerts indefinitely.
4. `days_until_stockout` returns null (not zero, not an error) when `avg_daily_sales = 0`. This can happen if a product had sales in the window (passed the `HAVING` filter) but the math rounds down to zero. Returning null signals "cannot calculate" rather than "stockout today."
5. When a product has multiple suppliers, the one with `is_primary = TRUE` is surfaced. Ties are broken alphabetically. If no supplier exists, `supplier` is null in the response.
6. Bundle products are treated identically to standard products for alerting purposes. Whether selling a bundle should deduct from component stock is a separate fulfilment concern, out of scope here.
7. Stock cannot go negative (`CHECK (quantity >= 0)` in schema). Backorders are not modelled.
8. The `company_id` in the URL is an integer. String slugs would need a different lookup.
