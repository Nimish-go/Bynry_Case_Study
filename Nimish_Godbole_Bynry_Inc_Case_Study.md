# Case Study: StockFlow Inventory Management System (B2B Saas)

### Descriptive Solution & Reasoning

## Part 1: Code Review And Debugging

### Problem Overview

The product creation and inventory initialization are handled by the specified endpoint. Although it seems to work, it has a number of technical, transactional and business logic issues.

### Issues Identified

1. **No Input Validation**

- The API assumes that all necessary and important fields are already present. Missing or malformed inputs cause runtime errors.

- ***Impact***: Frequent 500 errors and poor API consumer experience.

2. **SKU uniqueness**

- The endpoint allows duplicate SKU's even though SKU's must be unique across the platform.

- ***Impact***: Inventory collisions, incorrect reporting, and broken supplier integrations.

3. **Incorrect Data Modelling**:

- Products are directly associated with a warehouse, contradicting the requirement that products can exist in multiple warehouses.

- ***Impact***: System becomes non-scalable and requires major refactoring later.

4. **Non-atomic database operations**

- Two seperate ```commit()``` calls are used.

- ***Impact***: Partial writes - products can exist without inventory if the second operation fails.

5. **No Error Handling or Rollback**

- All failures return success unless the app crashes.

- ***Impact***: Silent data corruption and operational risk.

### Correct Implementation

```python

    from decimal import Decimal
    from sqlalchemy.exc import IntegrityError

    @app.route('/api/products', methods=['POST'])
    def create_product():
        data = request.json or {}
    
        # Basic validation
        for field in ['name', 'sku', 'price']:
            if field not in data:
                return {"error": f"{field} is required"}, 400
    
        try:
            with db.session.begin():  # Atomic transaction
                product = Product(
                    name=data['name'],
                    sku=data['sku'],
                    price=Decimal(str(data['price']))
                )
                db.session.add(product)
                db.session.flush()  # Ensures product.id is available
    
                # Optional inventory initialization
                if data.get('warehouse_id') and data.get('initial_quantity') is not None:
                    inventory = Inventory(
                        product_id=product.id,
                        warehouse_id=data['warehouse_id'],
                        quantity=int(data['initial_quantity'])
                    )
                    db.session.add(inventory)
    
            return {
                "message": "Product created successfully",
                "product_id": product.id
            }, 201
    
        except IntegrityError:
            db.session.rollback()
            return {"error": "SKU must be unique"}, 409
        except Exception:
            db.session.rollback()
            return {"error": "Failed to create product"}, 500

```

### Why This Works?

- Ensures atomic writes
- Supports multi-warehouse inventory
- Returns Meaningful API errors

## Part 2: Database Design

### Design Goals

- Support multi - warehouse inventory
- Maintain full auditability of stock changes
- Enable supplier - driven reordering
- Allow bundled products
- Scale cleanly for B2B SaaS usage

### Schema Design (Core Tables)

***Company Table***

```sql
companies(
    id UUID PRIMARY KEY,
    name VARCHAR,
    created_at TIMESTAMP
)
```

***Warehouse Table***

```sql
warehouses(
    id UUID PRIMARY KEY,
    company_id UUID REFERENCES companies(id),
    name VARCHAR,
    location VARCHAR
)
```

***Products Table***

```sql
products (
    id UUID PRIMARY KEY,
    company_id UUID,
    name VARCHAR,
    sku VARCHAR UNIQUE,
    price DECIMAL(10,2),
    product_type ENUM('standard', 'bundle')
)
```

***Inventory Table***

```sql
inventory (
    id UUID PRIMARY KEY,
    product_id UUID REFERENCES products(id),
    warehouse_id UUID REFERENCES warehouses(id),
    quantity INT,
    UNIQUE(product_id, warehouse_id)
)
```

***Inventory Movements Table***

```sql
inventory_movements (
    id UUID PRIMARY KEY,
    product_id UUID REFERENCES products(id),
    warehouse_id UUID REFERENCES warehouses(id),
    change INT,
    reason VARCHAR,
    created_at TIMESTAMP
)
```

***Suppliers Table***

```sql
suppliers (
    id UUID PRIMARY KEY,
    name VARCHAR,
    contact_email VARCHAR
)
```

***Product-Supplier Matching***

```sql
product_suppliers (
    product_id UUID PRIMARY KEY,
    supplier_id UUID REFERENCES suppliers(id),
)
```

***Product Bundles***

```sql
product_bundles (
    bundle_id UUID PRIMARY KEY,
    child_product_id UUID,
    quantity INT
)
```

### Design Rationale

- **Inventory table** enables multiple warehouses per product

- **Movement table** provides full stock traceability

- **Composite unique constraints** prevent duplicate inventory rows

- **Bundle mapping** supports nested product composition

- **Supplier mapping** enables flexible sourcing

### Missing Requirements

- Is ***SKU*** uniqueness global or company specific?

- How are ***returns*** and ***transfers*** handled?

- Are ***suppliers*** warehouse specific?

- What defines ***recent sales activity***?

## Part 3: Low Stock Alerts API

### Endpoint

```bash
GET /api/companies/{company_id}/alerts/low-stock
```

### Business Logic

For a given company, the system evaluates inventory across all warehouses.
Alerts are generated only when:

- Current stock is below the product's low-stock threshold

- The product has recent sales activity

- Stock Depletion can be meaningfully predicted

Each alert includes supplier details to allow immediate reordering.

### Implementation

```python
@app.route('/api/companies/<company_id>/alerts/low-stock')
def low_stock_alerts(company_id):
    alerts = []

    records = (
        db.session.query(Inventory, Product, Warehouse)
        .join(Product)
        .join(Warehouse)
        .filter(Warehouse.company_id == company_id)
        .all()
    )

    for inventory, product, warehouse in records:
        threshold = product.low_stock_threshold

        if inventory.quantity >= threshold:
            continue

        recent_sales = get_recent_sales(product.id, days=30)
        if recent_sales <= 0:
            continue

        avg_daily_sales = recent_sales / 30
        days_until_stockout = int(inventory.quantity / avg_daily_sales)

        supplier = get_primary_supplier(product.id)

        alerts.append({
            "product_id": product.id,
            "product_name": product.name,
            "sku": product.sku,
            "warehouse_id": warehouse.id,
            "warehouse_name": warehouse.name,
            "current_stock": inventory.quantity,
            "threshold": threshold,
            "days_until_stockout": days_until_stockout,
            "supplier": {
                "id": supplier.id,
                "name": supplier.name,
                "contact_email": supplier.contact_email
            } if supplier else None
        })

    return {
        "alerts": alerts,
        "total_alerts": len(alerts)
    }
```

### Edge Cases Handled

- No recent sales -> no alert

- Zero sales velocity -> avoids division errors

- Missing Supplier -> safe fallback

- Multiple warehouses -> aggregated correctly

### Summary
This solution demonstrates

- Strong debugging and transactional awareness

- Real world inventory modeling

- Thoughtful API Design

- Clear separation of business rules and data access

- Awareness of scalability, auditability, and data integrity

It balances **engineering best practices** with **business practically**, which is critically for B2B SaaS systems like ***Stockflow***.

To access this in google drive click [Google Drive Doc Link](https://docs.google.com/document/d/1fcyqXunsYF-oNkGzgPcsQHwwOcrcCJtDQSMpOgS5xuQ/edit?usp=sharing).
