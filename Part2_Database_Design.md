# Part 2: Database Design

## Schema Design

```sql
-- Companies table
CREATE TABLE companies (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    address TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    INDEX idx_name (name),
    INDEX idx_email (email)
);

-- Warehouses table
CREATE TABLE warehouses (
    id INT PRIMARY KEY AUTO_INCREMENT,
    company_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    address TEXT NOT NULL,
    city VARCHAR(100),
    state VARCHAR(50),
    zip_code VARCHAR(20),
    country VARCHAR(100),
    capacity_units INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (company_id) REFERENCES companies(id) ON DELETE CASCADE,
    INDEX idx_company_id (company_id),
    INDEX idx_name (name),
    UNIQUE KEY unique_warehouse_per_company (company_id, name)
);

-- Product Types table (for different low-stock thresholds)
CREATE TABLE product_types (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    low_stock_threshold INT NOT NULL,
    default_reorder_quantity INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Products table
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    sku VARCHAR(100) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(12, 2) NOT NULL CHECK (price > 0),
    product_type_id INT NOT NULL DEFAULT 1,
    is_bundle BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_by_company_id INT NOT NULL,
    FOREIGN KEY (product_type_id) REFERENCES product_types(id),
    FOREIGN KEY (created_by_company_id) REFERENCES companies(id),
    INDEX idx_sku (sku),
    INDEX idx_product_type_id (product_type_id),
    INDEX idx_is_bundle (is_bundle)
);

-- Bundle Products mapping (for composite products)
CREATE TABLE bundle_items (
    id INT PRIMARY KEY AUTO_INCREMENT,
    bundle_product_id INT NOT NULL,
    item_product_id INT NOT NULL,
    quantity_required INT NOT NULL CHECK (quantity_required > 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (bundle_product_id) REFERENCES products(id) ON DELETE CASCADE,
    FOREIGN KEY (item_product_id) REFERENCES products(id) ON DELETE CASCADE,
    UNIQUE KEY unique_bundle_item (bundle_product_id, item_product_id),
    INDEX idx_bundle_product_id (bundle_product_id)
);

-- Suppliers table
CREATE TABLE suppliers (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    contact_person VARCHAR(255),
    address TEXT,
    city VARCHAR(100),
    state VARCHAR(50),
    country VARCHAR(100),
    payment_terms VARCHAR(100),
    lead_time_days INT DEFAULT 7,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    INDEX idx_name (name),
    INDEX idx_email (email)
);

-- Supplier-Product relationships
CREATE TABLE supplier_products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    supplier_id INT NOT NULL,
    product_id INT NOT NULL,
    supplier_sku VARCHAR(100),
    cost_price DECIMAL(12, 2) NOT NULL CHECK (cost_price > 0),
    minimum_order_quantity INT DEFAULT 1,
    lead_time_days INT,
    last_price_update TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (supplier_id) REFERENCES suppliers(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    UNIQUE KEY unique_supplier_product (supplier_id, product_id),
    INDEX idx_supplier_id (supplier_id),
    INDEX idx_product_id (product_id)
);

-- Inventory tracking (current stock levels)
CREATE TABLE inventory (
    id INT PRIMARY KEY AUTO_INCREMENT,
    product_id INT NOT NULL,
    warehouse_id INT NOT NULL,
    quantity INT NOT NULL DEFAULT 0 CHECK (quantity >= 0),
    reserved_quantity INT DEFAULT 0 CHECK (reserved_quantity >= 0),
    available_quantity INT GENERATED ALWAYS AS (quantity - reserved_quantity) STORED,
    last_count_date DATE,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id) ON DELETE CASCADE,
    UNIQUE KEY unique_product_warehouse (product_id, warehouse_id),
    INDEX idx_product_id (product_id),
    INDEX idx_warehouse_id (warehouse_id),
    INDEX idx_low_stock (quantity)
);

-- Inventory movement history (audit trail)
CREATE TABLE inventory_movements (
    id INT PRIMARY KEY AUTO_INCREMENT,
    inventory_id INT NOT NULL,
    movement_type ENUM('purchase', 'sale', 'adjustment', 'transfer', 'return') NOT NULL,
    quantity_change INT NOT NULL,
    reference_id VARCHAR(100),
    reference_type VARCHAR(50),
    notes TEXT,
    created_by VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (inventory_id) REFERENCES inventory(id) ON DELETE CASCADE,
    INDEX idx_inventory_id (inventory_id),
    INDEX idx_created_at (created_at),
    INDEX idx_movement_type (movement_type)
);

-- Sales transactions (for recent activity tracking)
CREATE TABLE sales (
    id INT PRIMARY KEY AUTO_INCREMENT,
    warehouse_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity_sold INT NOT NULL CHECK (quantity_sold > 0),
    sale_price DECIMAL(12, 2) NOT NULL,
    sale_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    customer_id INT,
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id),
    FOREIGN KEY (product_id) REFERENCES products(id),
    INDEX idx_product_id (product_id),
    INDEX idx_warehouse_id (warehouse_id),
    INDEX idx_sale_date (sale_date),
    INDEX idx_recent_sales (warehouse_id, product_id, sale_date)
);

-- Company-Supplier relationships
CREATE TABLE company_suppliers (
    id INT PRIMARY KEY AUTO_INCREMENT,
    company_id INT NOT NULL,
    supplier_id INT NOT NULL,
    is_preferred BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (company_id) REFERENCES companies(id) ON DELETE CASCADE,
    FOREIGN KEY (supplier_id) REFERENCES suppliers(id) ON DELETE CASCADE,
    UNIQUE KEY unique_company_supplier (company_id, supplier_id),
    INDEX idx_company_id (company_id),
    INDEX idx_supplier_id (supplier_id)
);
```

## Key Design Decisions

### 1. **Separation of Concerns**
- **Products** - Global product catalog (SKU, price, metadata)
- **Inventory** - Location-specific stock levels (quantity at each warehouse)
- **Inventory_movements** - Audit trail of all changes

**Reasoning:** Products can exist in multiple warehouses. Separating product data from quantity allows tracking across locations without duplication.

### 2. **Product Types Table**
- Stores low-stock threshold by category
- Allows different thresholds for different product categories
- Can be updated without changing individual products

**Reasoning:** Business rule requires different thresholds by product type. Centralized configuration reduces duplication and enables quick policy changes.

### 3. **Bundle Products**
- Separate `bundle_items` table for many-to-many relationship
- Bundles are products themselves (can be ordered)
- Items can belong to multiple bundles

**Reasoning:** Flexible design supports complex product hierarchies. A "Kit" bundle might contain items that also exist as standalone products.

### 4. **Supplier Relationships**
- **Suppliers** - Supplier master data
- **Supplier_products** - Pricing and lead times per product
- **Company_suppliers** - Which suppliers each company uses

**Reasoning:** Same product may have different suppliers with different pricing/lead times. Supports multi-sourcing strategy.

### 5. **Inventory Reservations**
- `reserved_quantity` field for items allocated to orders but not shipped
- `available_quantity` calculated as generated column

**Reasoning:** Prevents overselling by distinguishing total stock from available stock. Real-time calculation ensures accuracy.

### 6. **Calculated Column**
- `available_quantity = quantity - reserved_quantity` is a generated column
- Automatically updated, always consistent

**Reasoning:** Improves query performance and prevents calculation errors in application code.

### 7. **Indexes**
- Composite index on `inventory(warehouse_id, product_id, quantity)` for low-stock queries
- Index on `sales.sale_date` for recent activity filtering
- Unique constraints where data must be unique

**Reasoning:** Low-stock alert queries will frequently filter by warehouse and sort by quantity. Recent sales queries need date indexes.

### 8. **Audit Trail**
- `inventory_movements` table tracks all changes
- Includes who made the change and why
- Supports historical analysis

**Reasoning:** Compliance and debugging - need to track inventory adjustments for discrepancy investigation.

---

## Questions for Product Team (Gaps Identified)

1. **Reorder Point Strategy**
   - Is low-stock threshold fixed per product type, or should it vary by warehouse?
   - Should lead time affect the reorder point calculation?
   - Example: If lead time is 14 days and daily sales are 5 units, should threshold be ~70?

2. **Multi-Company Support**
   - Can suppliers provide to multiple companies?
   - Should pricing be company-specific or supplier-wide?
   - Can companies share warehouse locations, or is each warehouse company-exclusive?

3. **Bundle/Kit Management**
   - When a bundle is low stock, how do we check if component items have sufficient stock?
   - Should we create "phantom" inventory for bundles based on component availability?

4. **Sales Data Requirements**
   - How far back should we check for "recent sales activity"? (7 days? 30 days? Configurable?)
   - What's the minimum frequency of sales to consider a product "active"?

5. **Warehouse Transfers**
   - Can inventory be transferred between warehouses?
   - Should transfers show up in low-stock alerts (subtract from source, add to destination)?

6. **Backorder Handling**
   - How are backordered items handled?
   - Should they reserve inventory that's on order from suppliers?

7. **Seasonal Products**
   - Are some products seasonal?
   - Should thresholds change seasonally?

8. **Return/Defect Handling**
   - How are returned items tracked?
   - Should defective items be separate from usable inventory?

9. **Multi-Currency Support**
   - Should pricing support multiple currencies per company?
   - Is there a single reporting currency?

10. **Alert Preferences**
    - Should companies configure which low-stock alerts they receive?
    - Should alerts be sent proactively or only on-demand?
    - What's the escalation path if stock stays low?
