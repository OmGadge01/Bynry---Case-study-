# Bynry Case-study (Pdf can be refered for better explanation)
# Name – Om Gadge

---

# Part 1 – Code Review & Debugging

```python
@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json
```

## Problem 1: No Input Validation

### Solution:
- Check required fields (name, sku)
- Ensure price > 0
- Convert price to decimal
- Use try-catch for error handling

### Correct Code:
```python
required_fields = ['name', 'sku']
for field in required_fields:
    if field not in data:
        return {"error": f"{field} is required"}, 400

price = Decimal(str(data.get('price', 0)))
initial_quantity = int(data.get('initial_quantity', 0))
warehouse_id = data.get('warehouse_id')

if price < 0:
    return {"error": "Price cannot be negative"}, 400

if initial_quantity < 0:
    return {"error": "Quantity cannot be negative"}, 400
```

### Impact:
- API becomes unreliable (500 errors)
- Bad data corrupts system

---

## Problem 2: SKU Not Unique

### Solution:
```python
existing = Product.query.filter_by(sku=data['sku']).first()
if existing:
    return {"error": "SKU already exists"}, 409
```

---

## Problem 3: Incorrect Business Logic (Warehouse Handling)

### Solution:
```python
warehouse = None
if warehouse_id:
    warehouse = Warehouse.query.get(warehouse_id)
    if not warehouse:
        return {"error": "Invalid warehouse_id"}, 400
```

### Impact:
- Breaks multi-warehouse design
- Tight coupling

---

## Problem 4: No Transaction Safety

### Solution:
```python
with db.session.begin():
    product = Product(
        name=data['name'],
        sku=data['sku'],
        price=price
    )
    db.session.add(product)
    db.session.flush()

    if warehouse_id:
        inventory = Inventory(
            product_id=product.id,
            warehouse_id=warehouse_id,
            quantity=initial_quantity
        )
        db.session.add(inventory)
```

### Impact:
- Prevents partial data writes
- Ensures consistency

---

## Final List of Errors

1. No Input Validation  
2. SKU Uniqueness Not Enforced  
3. No Transaction Safety  
4. Incorrect Warehouse Logic  
5. No Error Handling  
6. Float used for price  
7. Missing initial_quantity handling  
8. No warehouse validation  
9. No idempotency  
10. Weak API response  

---

# Part 2 – Database Design

## Tables

### Companies
```sql
companies (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
```

### Warehouses
```sql
warehouses (
    id SERIAL PRIMARY KEY,
    company_id INT REFERENCES companies(id),
    name VARCHAR(255),
    location VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
```

### Products
```sql
products (
    id SERIAL PRIMARY KEY,
    company_id INT REFERENCES companies(id),
    name VARCHAR(255) NOT NULL,
    sku VARCHAR(100) UNIQUE NOT NULL,
    price DECIMAL(10,2),
    is_bundle BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
```

### Inventory
```sql
inventory (
    id SERIAL PRIMARY KEY,
    product_id INT REFERENCES products(id),
    warehouse_id INT REFERENCES warehouses(id),
    quantity INT DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(product_id, warehouse_id)
)
```

### Inventory Logs
```sql
inventory_logs (
    id SERIAL PRIMARY KEY,
    product_id INT,
    warehouse_id INT,
    change_type VARCHAR(50),
    quantity_changed INT,
    previous_quantity INT,
    new_quantity INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
```

### Suppliers
```sql
suppliers (
    id SERIAL PRIMARY KEY,
    company_id INT REFERENCES companies(id),
    name VARCHAR(255),
    contact_info TEXT
)
```

### Product Suppliers
```sql
product_suppliers (
    id SERIAL PRIMARY KEY,
    product_id INT REFERENCES products(id),
    supplier_id INT REFERENCES suppliers(id),
    cost_price DECIMAL(10,2)
)
```

### Bundle Items
```sql
bundle_items (
    id SERIAL PRIMARY KEY,
    bundle_product_id INT REFERENCES products(id),
    component_product_id INT REFERENCES products(id),
    quantity INT
)
```

---

## Design Decisions

- Separate inventory table → supports multi-warehouse  
- Unique SKU → prevents duplicates  
- Composite index → avoids duplicate inventory rows  
- Inventory logs → audit + tracking  
- Bundle design → flexible product composition  
- Decimal price → financial accuracy  

---

## Questions to Ask

1. SKU unique globally or per company?  
2. Product without warehouse allowed?  
3. Need batch/expiry tracking?  
4. Suppliers global or per company?  
5. Need purchase orders?  

---

# Part 3 – API Implementation

## Assumptions

- low_stock_threshold exists  
- Recent sales = last 30 days  
- inventory per warehouse  
- supplier exists  

---

## Implementation

```python
@app.route('/api/companies/<int:company_id>/alerts/low-stock', methods=['GET'])
def get_low_stock_alerts(company_id):
    try:
        alerts = []
        last_30_days = datetime.utcnow() - timedelta(days=30)

        products = Product.query.filter_by(company_id=company_id).all()

        for product in products:
            recent_sales = db.session.query(OrderItem).join(Order).filter(
                Order.company_id == company_id,
                OrderItem.product_id == product.id,
                Order.created_at >= last_30_days
            ).count()

            if recent_sales == 0:
                continue

            inventories = Inventory.query.filter_by(product_id=product.id).all()

            for inv in inventories:
                if inv.quantity is None:
                    continue

                threshold = product.low_stock_threshold or 0

                if inv.quantity < threshold:
                    avg_daily_sales = recent_sales / 30 if recent_sales else 0
                    days_until_stockout = int(inv.quantity / avg_daily_sales) if avg_daily_sales > 0 else None

                    warehouse = Warehouse.query.get(inv.warehouse_id)

                    supplier = db.session.query(Supplier).join(ProductSupplier).filter(
                        ProductSupplier.product_id == product.id
                    ).first()

                    alerts.append({
                        "product_id": product.id,
                        "product_name": product.name,
                        "sku": product.sku,
                        "warehouse_id": inv.warehouse_id,
                        "warehouse_name": warehouse.name if warehouse else None,
                        "current_stock": inv.quantity,
                        "threshold": threshold,
                        "days_until_stockout": days_until_stockout,
                        "supplier": {
                            "id": supplier.id if supplier else None,
                            "name": supplier.name if supplier else None,
                            "contact_email": supplier.contact_email if supplier else None
                        }
                    })

        return {
            "alerts": alerts,
            "total_alerts": len(alerts)
        }, 200

    except Exception as e:
        return {"error": str(e)}, 500
```

---

## Edge Cases

- No recent sales → skip  
- No supplier → null values  
- Division by zero handled  
- Missing threshold → default 0  
- Null quantity skipped  

---

## Approach

- Filter products by company  
- Check recent sales  
- Loop inventory per warehouse  
- Compare with threshold  
- Calculate stockout days  
- Attach supplier info  
- Return alerts  
