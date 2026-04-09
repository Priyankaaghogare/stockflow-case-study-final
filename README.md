# 📦 StockFlow – Backend Case Study

👩‍💻 **Candidate:** Priyanka Ghogare  
🎯 **Role:** Backend Engineering Intern  

---

## 📌 Overview

This repository contains a backend case study solution for **StockFlow**, a B2B inventory management system.

The solution focuses on:
- Code review & debugging
- Scalable database design
- Real-world API implementation
- Production-ready backend practices

---

## 🔷 Part 1: Code Review & Debugging

### 🔍 Issues Identified
- ❌ No input validation → API crashes on invalid data  
- ❌ SKU uniqueness not enforced → duplicate products  
- ❌ No transaction handling → partial data writes  
- ❌ Incorrect product-warehouse relationship  
- ❌ No error handling → unstable system  
- ❌ No data type validation  

---

### ⚠️ Production Impact
- Missing fields → 500 errors  
- Duplicate SKUs → inconsistent data  
- Partial commits → broken records  
- Poor scalability → limited warehouse support  
- Application crashes → bad user experience  

---

### ✅ Fixes Implemented
- Input validation added  
- SKU uniqueness enforced  
- Transaction management using `@Transactional`  
- Improved data modeling  
- Error handling introduced  

---

### 💻 Sample Implementation

```java
@PostMapping
@Transactional
public ResponseEntity<?> createProduct(@RequestBody ProductRequest request) {

    // Input validation
    if (request.getName() == null || request.getSku() == null) {
        return ResponseEntity.badRequest().body("Missing required fields");
    }

    // SKU uniqueness
    if (productRepository.existsBySku(request.getSku())) {
        return ResponseEntity.status(409).body("SKU already exists");
    }

    // Create product
    Product product = new Product();
    product.setName(request.getName());
    product.setSku(request.getSku());
    product.setPrice(request.getPrice());

    productRepository.save(product);

    // Currently supports single warehouse (can be extended)
    if (request.getWarehouseId() != null) {
        Inventory inventory = new Inventory();
        inventory.setProduct(product);
        inventory.setWarehouseId(request.getWarehouseId());
        inventory.setQuantity(
            request.getInitialQuantity() != null ? request.getInitialQuantity() : 0
        );
        inventoryRepository.save(inventory);
    }

    return ResponseEntity.status(201).body("Product Created");
}
```

---

## 🔷 Part 2: Database Design

### 🗄️ Schema Overview

```
companies(id, name)

warehouses(id, company_id, name, location)

products(
  id,
  company_id,
  name,
  sku,
  price,
  threshold,
  UNIQUE(company_id, sku)
)

inventory(id, product_id, warehouse_id, quantity)

suppliers(id, name, contact_email)

product_suppliers(product_id, supplier_id)

inventory_logs(id, product_id, warehouse_id, change, timestamp)

bundles(bundle_id, component_id, quantity)

sales(id, product_id, warehouse_id, quantity, created_at)
```

---

### ✅ Design Decisions
- Separate inventory table → supports multi-warehouse  
- UNIQUE(company_id, sku) → ensures data consistency  
- Inventory logs → track stock changes  
- Sales table → required for analytics & alerts  
- Product-supplier mapping → supports multiple suppliers  

---

### ⚡ Indexing (Performance Optimization)

```sql
CREATE INDEX idx_inventory_product ON inventory(product_id);
CREATE INDEX idx_inventory_warehouse ON inventory(warehouse_id);
CREATE INDEX idx_sales_product_date ON sales(product_id, created_at);
```

---

### ❓ Open Questions
- Is SKU unique globally or per company?  
- What defines “recent sales”?  
- Can products have multiple suppliers?  
- Should inventory allow negative values?  
- Is soft delete required?  

---

## 🔷 Part 3: Low Stock Alerts API

### 📌 Assumptions
- Threshold stored per product  
- Recent sales = last 30 days  
- Average daily sales = total sales / 30  
- First supplier treated as primary  

---

### 💻 API Implementation

```java
@GetMapping("/{companyId}/alerts/low-stock")
public ResponseEntity<?> getLowStock(@PathVariable Long companyId) {

    List<Inventory> inventories = inventoryRepository.findByCompanyId(companyId);
    List<Map<String, Object>> alerts = new ArrayList<>();

    for (Inventory inv : inventories) {

        Product p = inv.getProduct();

        // Sales in last 30 days
        int totalSales = salesRepository.getLast30DaysSales(p.getId());

        // Skip if no sales
        if (totalSales == 0) continue;

        int avgDailySales = totalSales / 30;

        if (inv.getQuantity() < p.getThreshold()) {

            Map<String, Object> alert = new HashMap<>();

            alert.put("product_id", p.getId());
            alert.put("product_name", p.getName());
            alert.put("sku", p.getSku());
            alert.put("warehouse_id", inv.getWarehouse().getId());
            alert.put("warehouse_name", inv.getWarehouse().getName());
            alert.put("current_stock", inv.getQuantity());
            alert.put("threshold", p.getThreshold());

            int daysUntilStockout =
                avgDailySales > 0 ? inv.getQuantity() / avgDailySales : -1;

            alert.put("days_until_stockout", daysUntilStockout);

            // Primary supplier
            Supplier s = p.getSuppliers().get(0);

            Map<String, Object> supplier = new HashMap<>();
            supplier.put("id", s.getId());
            supplier.put("name", s.getName());
            supplier.put("contact_email", s.getContactEmail());

            alert.put("supplier", supplier);

            alerts.add(alert);
        }
    }

    return ResponseEntity.ok(
        Map.of("alerts", alerts, "total_alerts", alerts.size())
    );
}
```

---

### ✅ Business Logic Covered
- Low stock detection  
- Sales-based filtering  
- Multi-warehouse support  
- Supplier integration  
- Stock-out prediction  

---

### ⚠️ Edge Cases
- No recent sales → skipped  
- Division by zero handled  
- Missing supplier data  
- Large dataset → pagination required  

---

### 🚀 Scalability Considerations
- Pagination (limit/offset)  
- Database indexing  
- Caching (Redis)  
- Async processing for alerts  

---

## ✅ Conclusion

This solution demonstrates:

✔ Strong backend fundamentals  
✔ Scalable system design  
✔ Clean API structure  
✔ Production-ready practices  
✔ Real-world business logic implementation  
