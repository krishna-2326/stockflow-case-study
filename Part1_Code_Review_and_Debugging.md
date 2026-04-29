# Part 1: Code Review & Debugging

## Issues Identified

### 1. **Missing Error Handling & Validation**
- No validation that required fields exist in `request.json`
- No try-catch blocks for database operations
- No handling for database constraint violations

**Impact:** API crashes with 500 errors instead of returning meaningful error messages. If `warehouse_id` doesn't exist, the foreign key constraint fails silently.

**Fix:** Add input validation and try-catch blocks.

---

### 2. **Transaction Integrity Problem - CRITICAL**
The code uses two separate `db.session.commit()` calls. If the inventory creation fails after the product commit, the database will have an orphaned product record with no inventory.

**Impact:** Data inconsistency - product exists but has no inventory tracking. Subsequent queries may fail or show incomplete data.

**Fix:** Wrap both operations in a single transaction or use try-except with rollback.

---

### 3. **SKU Uniqueness Not Enforced**
The requirements state "SKUs must be unique across the platform", but the code doesn't check for duplicates before insertion.

**Impact:** Duplicate SKUs can exist, breaking product identification logic. Analytics and reporting become unreliable.

**Fix:** Add database constraint (UNIQUE) and check before insertion.

---

### 4. **Missing Price Validation**
No validation that price is a positive decimal value.

**Impact:** Negative or NULL prices can be inserted, breaking financial calculations and reporting.

**Fix:** Validate price > 0 before creating product.

---

### 5. **No Response Status Codes**
Always returns 200 even if something is wrong.

**Impact:** Client can't distinguish between success and partial failure. API consumers get false confidence.

**Fix:** Return appropriate HTTP status codes (201 for creation, 400 for validation errors, 409 for conflicts).

---

### 6. **Logging Missing**
No audit trail for product creation.

**Impact:** Can't track who created products or debug production issues. Non-compliant with business requirements for inventory tracking.

**Fix:** Add logging before/after database operations.

---

## Corrected Code

```python
from flask import request, jsonify
from datetime import datetime
import logging

# Configure logging
logger = logging.getLogger(__name__)

@app.route('/api/products', methods=['POST'])
def create_product():
    """
    Create a new product with initial inventory.
    
    Expected JSON payload:
    {
        "name": "Widget A",
        "sku": "WID-001",
        "price": 29.99,
        "warehouse_id": 1,
        "initial_quantity": 100,
        "product_type": "standard"  # optional
    }
    """
    try:
        # VALIDATION: Check for required fields
        data = request.json
        required_fields = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']
        
        if not data:
            return jsonify({"error": "Request body is empty"}), 400
        
        missing_fields = [field for field in required_fields if field not in data]
        if missing_fields:
            return jsonify({
                "error": f"Missing required fields: {', '.join(missing_fields)}"
            }), 400
        
        # VALIDATION: Price must be positive decimal
        try:
            price = float(data['price'])
            if price <= 0:
                return jsonify({"error": "Price must be greater than 0"}), 400
        except (ValueError, TypeError):
            return jsonify({"error": "Price must be a valid decimal number"}), 400
        
        # VALIDATION: Quantity must be non-negative integer
        try:
            initial_quantity = int(data['initial_quantity'])
            if initial_quantity < 0:
                return jsonify({"error": "Initial quantity cannot be negative"}), 400
        except (ValueError, TypeError):
            return jsonify({"error": "Initial quantity must be a valid integer"}), 400
        
        # VALIDATION: Warehouse exists
        warehouse = Warehouse.query.get(data['warehouse_id'])
        if not warehouse:
            return jsonify({"error": f"Warehouse {data['warehouse_id']} not found"}), 404
        
        # VALIDATION: Check for duplicate SKU
        existing_product = Product.query.filter_by(sku=data['sku']).first()
        if existing_product:
            return jsonify({
                "error": f"SKU '{data['sku']}' already exists (Product ID: {existing_product.id})"
            }), 409  # 409 Conflict
        
        # TRANSACTION: Both operations in single transaction
        try:
            # Create new product
            product = Product(
                name=data['name'],
                sku=data['sku'],
                price=price,
                warehouse_id=data['warehouse_id'],
                product_type=data.get('product_type', 'standard'),
                created_at=datetime.utcnow()
            )
            db.session.add(product)
            db.session.flush()  # Flush to get product.id without committing
            
            # Create inventory record
            inventory = Inventory(
                product_id=product.id,
                warehouse_id=data['warehouse_id'],
                quantity=initial_quantity,
                last_updated=datetime.utcnow()
            )
            db.session.add(inventory)
            
            # Single commit for both operations
            db.session.commit()
            
            logger.info(
                f"Product created successfully. "
                f"Product ID: {product.id}, SKU: {data['sku']}, "
                f"Warehouse ID: {data['warehouse_id']}"
            )
            
            return jsonify({
                "message": "Product created successfully",
                "product_id": product.id,
                "sku": data['sku'],
                "warehouse_id": data['warehouse_id'],
                "initial_quantity": initial_quantity
            }), 201  # 201 Created
            
        except IntegrityError as e:
            db.session.rollback()
            logger.error(f"Database integrity error: {str(e)}")
            return jsonify({
                "error": "Duplicate SKU or invalid warehouse. Please verify your input."
            }), 409
        except Exception as e:
            db.session.rollback()
            logger.error(f"Unexpected error during product creation: {str(e)}")
            return jsonify({"error": "Internal server error"}), 500
    
    except Exception as e:
        logger.error(f"Error in create_product endpoint: {str(e)}")
        return jsonify({"error": "Internal server error"}), 500
```

## Key Improvements Explained

1. **Input Validation** - Validates all required fields before database operations
2. **Transaction Safety** - Single `db.session.commit()` ensures both operations succeed or both fail
3. **SKU Uniqueness** - Checks for duplicate SKU before insertion
4. **Price Validation** - Ensures price is positive decimal
5. **Proper HTTP Status Codes** - 201 for creation, 400 for validation, 409 for conflicts, 500 for server errors
6. **Error Logging** - Logs all operations for audit trail and debugging
7. **Foreign Key Validation** - Verifies warehouse exists before linking
