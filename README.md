# StockFlow - B2B Inventory Management System Case Study

A comprehensive backend engineering case study covering code review, database design, and API implementation for a B2B inventory management platform.

## 📋 Project Overview

StockFlow is a SaaS platform that helps small businesses track products across multiple warehouses and manage supplier relationships. This case study covers three critical areas:

### Contents

1. **[Part 1: Code Review & Debugging](Part1_Code_Review_and_Debugging.md)**
   - Identifies 6 critical issues in a product creation API endpoint
   - Explains business and technical impacts
   - Provides production-ready corrected code
   - Key focus: Transaction integrity, validation, error handling

2. **[Part 2: Database Design](Part2_Database_Design.md)**
   - Complete SQL schema with 13 tables
   - Design decisions and rationale
   - Indexes and constraints for performance
   - 10 questions for product clarification

3. **[Part 3: API Implementation](Part3_API_Implementation.md)**
   - Low-stock alerts endpoint implementation
   - Complex SQL queries with joins and aggregations
   - Supplier lookup and reorder calculations
   - Edge case handling and performance optimization

## 🔑 Key Features Covered

- **Multi-warehouse inventory tracking** - Products stored across multiple locations
- **Real-time low-stock alerts** - Threshold-based notifications by product type
- **Supplier management** - Multi-sourcing with preferred supplier selection
- **Reorder intelligence** - Automatic calculation based on lead times and sales velocity
- **Audit trail** - Complete inventory movement history for compliance
- **Bundle products** - Support for composite products and kits

## 🏗️ Architecture Highlights

### Database Design
- Separation of concerns: Products, Inventory, Movements
- Normalized schema with proper foreign keys
- Composite indexes for query optimization
- Generated columns for calculated fields

### API Endpoints
- **POST /api/products** - Create products with validation
- **GET /api/companies/{company_id}/alerts/low-stock** - Retrieve low-stock alerts

### Error Handling
- Input validation at all layers
- Transaction rollback on failures
- Proper HTTP status codes (201, 400, 404, 409, 500)
- Comprehensive logging for debugging

## 📊 Business Rules Implemented

- SKU uniqueness across platform
- Price validation (must be positive decimal)
- Threshold varies by product type
- Only alert for products with recent sales
- Lead time-based reorder calculations
- Reserved inventory tracking

## 🚀 Usage

Each file contains:
- Problem analysis and explanation
- Production-ready code examples
- SQL schemas
- Design decisions with rationale
- Edge cases and testing recommendations
- Deployment checklist

## 💡 Technical Stack

- **Backend**: Python/Flask, Node.js/Express compatible
- **Database**: MySQL/PostgreSQL with InnoDB
- **ORM**: SQLAlchemy
- **API**: RESTful JSON responses

## 📝 Document Structure

```
casestudy/
├── Part1_Code_Review_and_Debugging.md    # Bug fixes & corrected code
├── Part2_Database_Design.md              # Schema design & decisions  
├── Part3_API_Implementation.md           # API implementation & optimization
├── Case_Study_Submission.md              # Combined submission version
├── README.md                             # This file
└── .gitignore                            # Git configuration
```

## 🎯 Key Takeaways

### Part 1: Code Quality
- Importance of transaction safety in multi-step operations
- Input validation prevents data corruption
- Proper HTTP status codes improve API usability
- Logging is essential for production support

### Part 2: Database Design
- Separation of concerns improves scalability
- Strategic indexing impacts query performance
- Asking clarifying questions prevents rework
- Audit trails support compliance and debugging

### Part 3: API Design
- Complex queries should be optimized, not N+1
- Business logic encoded in schema (product types, thresholds)
- Calculated fields (available_quantity) prevent calculation errors
- Flexible parameters (days_back) support different use cases

## 🔍 Edge Cases Handled

✅ No recent sales activity (skip from alerts)
✅ Division by zero (null handling for calculations)
✅ Reserved inventory (separate from available)
✅ Missing suppliers (skip products gracefully)
✅ Multiple warehouses per company (separate alerts)
✅ Different product types (category-specific thresholds)
✅ Concurrent updates (database isolation)
✅ Very high velocity items (priority sorting)

## 📋 Assumptions Made

- Recent sales activity = 30 days (configurable)
- Low-stock threshold from product_types table
- Safety stock buffer = 10% of lead time consumption
- Preferred suppliers marked in company_suppliers table
- UTC timezone for all timestamps
- Flask + SQLAlchemy stack

## 🚢 Production Deployment

Complete checklist included covering:
- Database indexes and configuration
- Connection pooling and performance
- Logging and error monitoring
- HTTPS and security
- Rate limiting and caching
- Backup and rollback procedures

## 👨‍💼 For Interviews

This case study demonstrates:
- Problem decomposition skills
- Ability to identify issues in production code
- Database design thinking
- API design and implementation
- Edge case identification
- Communication of technical decisions
- Understanding of business requirements

## 📞 Contact & Questions

Questions about the design or implementation?
- Review the "Questions for Product Team" section in Part 2
- Check the assumptions documented in Part 3
- Refer to edge case handling in each implementation

---

**Submission Date**: April 29, 2026
**Time Allocation**: 90 minutes (take-home), 30-45 minutes (live discussion)
**Status**: ✅ Complete and ready for review
