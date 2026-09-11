# E-Commerce, POS and Enterprise Inventory Platform

## Agent Implementation Plan

## 1. Objective

Build a configurable, multi-tenant commerce platform that can be sold to:

- Boutique and single-store businesses
- Startups and growing retailers
- Enterprise organizations with multiple companies, stores and warehouses

The platform must support three client applications:

1. Consumer e-commerce storefront
2. Point-of-sale application
3. Administrative backoffice

Use three core business services:

1. Catalog Service
2. Commerce Service
3. Inventory Service

IAM and Notification are separate platform services and are outside these three business boundaries.

The design must remain provider-independent. It should be possible to implement persistence through Blocks, Supabase, PostgreSQL, MongoDB or another provider without changing domain rules.

---

## 2. Product principles

- Use one platform and one codebase for all customer segments.
- Use feature entitlements, limits and configuration to define product editions.
- Do not create separate inventory engines for small and enterprise customers.
- Keep enterprise-grade consistency and auditing in the shared inventory foundation.
- Hide advanced features from customers that do not subscribe to them.
- Keep business logic independent from database and API providers.
- Do not let one service write directly to another service's database.
- Start with clear modules inside the three services; do not create additional microservices without a demonstrated scaling or ownership need.

---

## 3. System context

```text
Consumer Storefront ─┐
POS Application ─────┼──> API Gateway / BFF
Backoffice ──────────┘             │
                                  ├──> Catalog Service
                                  ├──> Commerce Service
                                  └──> Inventory Service

All services also integrate with IAM and Notification.
```

The API Gateway/BFF is an entry layer, not a business microservice. Separate BFF endpoints may be used for storefront, POS and backoffice when their response shapes differ significantly.

---

## 4. Service ownership

### 4.1 Catalog Service

Catalog Service owns what can be sold, how it is presented and its selling price.

#### Modules

- Products and variants
- Categories and brands
- Attributes and attribute groups
- Product media
- Product bundles and related products
- Prices and price lists
- Promotions and coupons
- Reviews and ratings
- Product publication by sales channel
- Search metadata and SEO

#### Owned data

```text
Products
ProductVariants
Categories
Brands
Attributes
AttributeGroups
ProductMedia
ProductBundles
RelatedProducts
PriceLists
Prices
Promotions
Coupons
Reviews
Collections
SalesChannelPublications
```

#### Rules

- A variant is the sellable unit.
- Catalog owns selling prices; Inventory owns inventory cost and valuation.
- Products can be published to one or more sales channels.
- Channel-specific prices, visibility and promotions must be supported.
- Catalog must not store authoritative stock quantities.
- Review verification must use purchase confirmation from Commerce.

#### Important events

```text
ProductCreated
ProductUpdated
ProductPublished
ProductArchived
VariantCreated
VariantUpdated
PriceChanged
PromotionStarted
PromotionEnded
ReviewSubmitted
ReviewApproved
```

### 4.2 Commerce Service

Commerce Service owns the customer purchase journey and commercial transaction.

#### Modules

- Commerce customer profile
- Customer addresses
- Favourites
- Shopping cart
- Checkout
- Online orders
- POS sales
- Payments and refunds
- Customer-facing shipments
- Returns and exchanges
- Receipts and invoices
- POS terminals and shifts

#### Owned data

```text
CommerceCustomers
CustomerAddresses
Favourites
Carts
CartItems
CheckoutSessions
Orders
OrderItems
POSTransactions
POSShifts
POSCashMovements
Payments
Refunds
ReturnRequests
Exchanges
CustomerShipments
Invoices
Receipts
OrderActivity
```

#### Rules

- IAM owns identity and authentication; Commerce stores only commerce-specific customer data.
- Order, payment, fulfillment and return states must be stored separately.
- Order lines must contain product, price, tax and discount snapshots.
- Payment requests and webhooks must be idempotent.
- Raw card data must never be stored.
- Commerce requests stock changes through Inventory APIs or commands.
- Commerce must not directly update inventory records.

#### Status groups

```text
OrderStatus: Draft, PendingPayment, Confirmed, Cancelled, Completed
PaymentStatus: Pending, Authorized, Paid, PartiallyRefunded, Refunded, Failed
FulfillmentStatus: Unallocated, Allocated, Picking, Packed, PartiallyShipped, Shipped, Delivered
ReturnStatus: None, Requested, Approved, Received, Inspected, Completed, Rejected
```

#### Important events

```text
CartCheckedOut
OrderCreated
OrderConfirmed
OrderCancelled
PaymentAuthorized
PaymentConfirmed
PaymentFailed
RefundCompleted
ReturnRequested
POSSaleCompleted
POSReturnCompleted
```

### 4.3 Inventory Service

Inventory Service is the primary selling capability. It owns all physical stock, warehouse, procurement and fulfillment operations.

#### Modules

- Inventory locations
- Warehouses and retail stores
- Zones and bins
- Inventory balances
- Inventory reservations
- Immutable inventory ledger
- Allocation
- Adjustments
- Stock transfers
- Suppliers
- Purchase orders
- Goods receiving
- Picking and packing
- Fulfillment
- Return inspection
- Stock counting
- Lots, batches and serial numbers
- Replenishment
- Inventory costing and valuation

#### Owned data

```text
InventoryLocations
Warehouses
WarehouseZones
Bins
InventoryBalances
InventoryReservations
InventoryMovements
StockAdjustments
StockTransfers
Suppliers
SupplierItems
PurchaseRequisitions
PurchaseOrders
GoodsReceipts
Fulfillments
PickLists
Packages
ReturnInspections
StockCounts
Lots
SerialNumbers
CostLayers
ReplenishmentRules
```

#### Important events

```text
InventoryReserved
InventoryReservationFailed
ReservationReleased
StockReceived
StockAdjusted
StockTransferred
GoodsReceived
OrderAllocated
PickingCompleted
ShipmentDispatched
ReturnInspected
LowStockDetected
OutOfStockDetected
```

---

## 5. Consumer storefront scope

### Home page

- Hero and promotional banners
- Featured categories and brands
- New arrivals
- Best sellers
- Recommended and recently viewed products
- Promotional collections
- Configurable content sections

### Product discovery

- Hierarchical categories
- Brand directory and details
- Product search and suggestions
- Faceted filters
- Sorting
- Pagination or infinite scrolling
- Availability filtering
- SEO metadata

### Product details

- Product and variant information
- Media gallery
- Specifications
- Price and promotion
- Variant selector
- Availability from Inventory
- Delivery or pickup estimates
- Reviews and ratings
- Favourites
- Related products

### Cart and checkout

- Guest and authenticated carts
- Cart merging after login
- Quantity validation
- Coupons and promotions
- Tax and shipping calculation
- Address selection
- Delivery method
- Payment
- Final price and inventory revalidation
- Idempotent order submission

### Customer account

- Profile and addresses
- Favourites
- Order history
- Order details and tracking
- Invoice download
- Cancellation where eligible
- Return and refund requests
- Reorder

---

## 6. POS scope

### Core sale operations

- Barcode scanning
- Fast product and SKU search
- Variant selection
- Store-level availability
- Add, remove and change line quantity
- Line and order discounts
- Customer attachment
- Hold and resume sale
- Cash, card and split payment
- Amount tendered and change calculation
- Receipt printing and email
- Returns and exchanges
- Void line and void sale with permissions
- Manager approval for restricted actions

### Store operations

- Register/terminal identification
- Open and close shift
- Opening cash balance
- Cash in/out
- End-of-day reconciliation
- Cashier activity
- Store stock lookup
- Store-to-store transfer request

### Offline operation

- Local product, price, tax and promotion cache
- Local transaction queue
- Stable terminal and transaction identifiers
- Synchronization status
- Retry and idempotency
- Conflict handling
- Configurable offline stock-selling policy

A POS terminal does not own inventory. All terminals at one store consume the same store inventory location.

---

## 7. Backoffice scope

### Catalog administration

- Products and variants
- Categories and brands
- Attributes and media
- Prices and price lists
- Promotions and coupons
- Reviews and moderation
- Channel publication
- Bulk import/export

### Commerce administration

- Online orders and POS sales
- Payment status
- Fulfillment status
- Cancellations
- Returns, exchanges and refunds
- Customer order history
- Invoices and receipts

### Inventory administration

- Warehouses, stores, zones and bins
- Stock overview by product, variant and location
- Reservations
- Immutable movement history
- Adjustments and approvals
- Transfers
- Suppliers and purchase orders
- Goods receiving
- Picking, packing and dispatch
- Return inspection
- Cycle counting
- Lots, serials and expiry
- Replenishment
- Valuation and stock ageing

### Dashboard and reports

- Sales and order metrics
- Available, reserved and incoming stock
- Low-stock and out-of-stock products
- Inventory valuation
- Inventory accuracy
- Stock ageing
- Pending transfers
- Open purchase orders
- Supplier performance
- Store and warehouse performance

---

## 8. Enterprise inventory requirements

### 8.1 Inventory dimensions

Maintain stock by:

```text
Tenant + Organization + InventoryLocation + Variant
+ optional Bin + optional Lot + optional SerialNumber
```

### 8.2 Quantity buckets

- OnHand
- Reserved
- AvailableToSell
- Incoming
- InTransit
- Damaged
- QualityHold
- Blocked
- Backordered

Default formula:

```text
AvailableToSell = OnHand - Reserved - Damaged - QualityHold - Blocked
```

Make the policy configurable without allowing inconsistent calculations in different applications.

### 8.3 Inventory ledger

Every stock change must create an immutable movement containing:

- Variant, location, bin, lot and serial references
- Signed quantity changes
- Balance before and after
- Unit cost
- Reference type and identifier
- Reason code
- Actor
- Timestamp
- Correlation and idempotency keys

Movements must not be edited or deleted. Corrections require reversing movements.

### 8.4 Reservations

Reservation lifecycle:

```text
Active -> Committed
Active -> Released
Active -> Expired
```

Requirements:

- Atomic availability check and reservation
- Expiration and background release
- Partial reservation
- Multi-location allocation
- Release after cancellation or payment failure
- Idempotent reserve, commit and release
- Backorder policies

### 8.5 Allocation

Support configurable strategies:

- Warehouse priority
- Nearest location
- Highest availability
- Lowest fulfillment cost
- Minimum shipment splitting
- Customer-selected store
- FIFO/FEFO lot selection

Allocation and reservation are separate concepts.

### 8.6 Transfers

Lifecycle:

```text
Draft -> Requested -> Approved -> Picking -> Dispatched
-> InTransit -> PartiallyReceived -> Received -> Closed
```

Track requested, approved, shipped, received, damaged and lost quantities. Dispatch moves stock to InTransit; receipt moves it to destination OnHand.

### 8.7 Procurement and receiving

- Supplier catalog and supplier SKU
- Purchase requisition
- Purchase-order approval
- Partial receiving
- Quality inspection
- Landed costs
- Supplier returns
- Supplier performance

Receipt must update balance and ledger reliably as one business transaction.

### 8.8 Stock counting

- Full physical counts
- Cycle counting
- Blind counts
- ABC classification
- Count assignment and recount
- Variance approval
- Adjustment posting
- Accuracy reporting

### 8.9 Traceability

- Lot and batch numbers
- Manufacturing and expiry dates
- Serial numbers
- Warranty
- Product recalls
- FIFO and FEFO

### 8.10 Costing

Support configurable methods:

- Weighted average
- FIFO
- Standard cost
- Specific identification

Preserve cost layers and historical cost. Do not value historical transactions using the current product cost.

### 8.11 Replenishment

- Reorder point and reorder quantity
- Safety stock
- Minimum/maximum levels
- Lead time
- Location-specific rules
- Purchase suggestions
- Transfer suggestions
- Low-stock and overstock alerts

---

## 9. Shared e-commerce and POS model

Both channels use the same catalog, commerce and inventory foundation.

Every transaction includes:

```text
TenantId
OrganizationId
SalesChannelId
SalesChannelType
LocationId when applicable
Currency
```

Sales channel types may include:

```text
Ecommerce
POS
MobileApp
Marketplace
Wholesale
```

### E-commerce inventory flow

```text
Checkout -> Reserve -> Pay -> Allocate -> Pick -> Pack -> Dispatch
```

### POS inventory flow

```text
Scan -> Validate store availability -> Pay -> Deduct store stock -> Issue receipt
```

Both flows write to the same Inventory ledger using different movement and reference types.

---

## 10. Multi-tenancy and product editions

### Tenant isolation

Every relevant record, query, cache key, event, file, search index and background job must carry tenant context.

Enforce isolation through:

- Token claims
- Application authorization
- Repository filters
- Database constraints or policies
- Event metadata
- Cache key prefixes
- Observability metadata

### Starter edition

For boutiques:

- One organization
- One or two locations
- Catalog
- Basic inventory and ledger
- POS
- Sales and returns
- Basic procurement and reports
- Limited users

### Growth edition

For startups and growing retailers:

- E-commerce and POS
- Multiple warehouses
- Reservations and allocation
- Transfers
- Suppliers and purchase orders
- Promotions
- Standard integrations
- Advanced reporting

### Enterprise edition

- Multi-company and many locations
- SSO and SCIM
- Advanced roles and approval workflows
- Lots, serials and expiry
- Cycle counting
- Advanced allocation and replenishment
- Inventory costing and valuation
- Offline POS
- Long audit retention
- Integration APIs and webhooks
- Dedicated deployment options
- SLA and disaster-recovery requirements

Use entitlements and limits. Disabled capabilities must be hidden in clients and rejected by backend authorization.

---

## 11. Provider-independent design

Define domain-level interfaces:

```text
ICatalogRepository
ICommerceRepository
IInventoryRepository
IUnitOfWork
IEventPublisher
IFileStorageProvider
ISearchProvider
IPaymentProvider
IShippingProvider
ITaxProvider
```

Provider adapters may include:

```text
Blocks
Supabase
MongoDB
PostgreSQL
External payment providers
External carrier providers
External tax providers
```

Domain and application layers must not reference provider-specific SDKs. If a provider cannot guarantee safe inventory operations, execute those operations in Inventory Service with a storage mechanism that supports atomicity and concurrency.

---

## 12. Consistency and integration

Use strong consistency for:

- Reservation and release
- POS sale deduction
- Shipment deduction
- Goods receipt
- Transfer dispatch and receipt
- Stock adjustment
- Payment/order state transitions

Use eventual consistency for:

- Search indexing
- Reporting projections
- Dashboards
- Recommendations
- Notifications

Use synchronous APIs when an immediate answer is required, and events for downstream processing.

Required reliability patterns:

- Database transactions where supported
- Atomic conditional updates
- Optimistic concurrency/version fields
- Idempotency keys
- Transactional outbox
- Consumer inbox/deduplication
- Retry policies
- Dead-letter queues
- Correlation IDs
- Reconciliation jobs

---

## 13. Security and access

Use IAM for authentication, users, roles and token issuance.

Backoffice roles should include:

- Administrator
- Catalog manager
- Order manager
- Warehouse manager
- Inventory operator
- Procurement manager
- Finance user
- Customer-support user
- Auditor

Support feature, action, location and sensitive-field permissions. Inventory cost, approvals, adjustments, refunds, voids and cash operations require explicit authorization and audit records.

---

## 14. Non-functional requirements

### Performance

- Paginate all large datasets.
- Prevent unbounded queries.
- Use appropriate compound indexes.
- Cache safe catalog reads.
- Avoid synchronous chains where events are sufficient.
- Establish measurable p50, p95 and p99 targets.

Initial targets:

```text
Catalog reads: p95 < 500 ms
Availability lookup: p95 < 200 ms
Reservation: p95 < 500 ms
POS checkout excluding provider latency: p95 < 1 second
```

### Reliability

- No negative stock unless an explicit backorder/negative-stock policy permits it.
- No duplicate movement for a retried operation.
- No lost events after a successful database commit.
- Automated backups and restore testing.
- Point-in-time recovery where supported.
- Reconciliation for stock, payments and events.

### Observability

- Structured logs
- Metrics
- Distributed traces
- Audit logs
- Slow-query monitoring
- Failed-event and dead-letter monitoring
- Stock inconsistency alerts
- Reservation-expiry monitoring
- Payment-webhook monitoring

### Security

- OIDC/OAuth2
- MFA for privileged users
- Least privilege
- Encryption in transit and at rest
- Secret management
- Rate limiting
- Input validation
- PII protection
- Payment security

---

## 15. Deployment models

Use the same application code for:

| Deployment | Target customer |
|---|---|
| Shared SaaS | Boutique and startup |
| Shared app with dedicated database | Growth and larger customers |
| Dedicated application and database | Enterprise |
| Private cloud | Regulated enterprise |
| On-premises | Customers requiring infrastructure control |

Avoid customer-specific forks. Use configuration, extension points and provider adapters.

---

## 16. Delivery roadmap

### Phase 1: Foundation

- Service skeletons and provider abstraction
- Tenant context and entitlements
- Catalog and variants
- Categories and brands
- Basic prices
- Storefront product discovery
- Basic POS sale
- Basic cart, checkout and orders
- Inventory locations, balances and ledger
- Backoffice foundations

### Phase 2: Core inventory

- Reservations
- Allocation
- Adjustments
- Multiple warehouses/stores
- Transfers
- Low-stock alerts
- Picking and packing
- Inventory dashboards

### Phase 3: Procurement and fulfillment

- Suppliers
- Purchase orders
- Goods receiving
- Partial receiving
- Shipments
- Partial fulfillment
- Customer returns and inspection
- Refund coordination

### Phase 4: Enterprise capabilities

- Bins and zones
- Lots, serials and expiry
- Cycle counting
- FIFO/FEFO
- Costing and valuation
- Advanced replenishment
- Approval workflows
- Offline POS synchronization
- Enterprise integrations
- Dedicated deployment support

---

## 17. Agent execution rules

1. Inspect the existing repository, conventions and instructions before changing code.
2. Produce a short gap analysis against this plan.
3. Do not implement the whole platform in one uncontrolled change.
4. Propose milestone-specific specifications and obtain scope confirmation before each milestone.
5. Preserve the three business-service boundaries.
6. Implement modules within services instead of creating more microservices.
7. Keep persistence and third-party integrations behind interfaces.
8. Do not let clients or services write directly to another service's database.
9. Include migrations, indexes, validation, authorization, observability and tests with each feature.
10. Use idempotency and concurrency controls for every stock-changing operation.
11. Add automated tests for inventory invariants and cross-service workflows.
12. Run build, lint, static analysis and tests before completing a milestone.
13. Document APIs, events, configuration and operational requirements.

---

## 18. Minimum acceptance criteria

The initial sellable version is complete when:

- A tenant can configure an organization, sales channel and inventory location.
- Backoffice users can manage catalog, prices and stock.
- Consumers can browse, add to cart, checkout and track orders.
- POS users can complete, hold, resume, void and return sales.
- Online and POS channels share authoritative stock.
- Inventory reservations prevent overselling.
- Every stock change produces one immutable, idempotent movement.
- Transfers, purchase orders and receiving maintain correct balances.
- Permissions and audit logs protect sensitive operations.
- Starter/Growth/Enterprise entitlements change available functionality.
- Persistence providers can be changed through adapters rather than domain rewrites.
- Critical workflows have integration and concurrency tests.
- Logs, metrics and traces allow failed transactions to be diagnosed.

---

## 19. Final architecture summary

```text
IAM Service
Notification Service
Catalog Service
Commerce Service
Inventory Service
API Gateway / BFF

Consumer Storefront
POS Application
Backoffice Application
```

- Catalog owns what can be sold and its selling price.
- Commerce owns customers' purchase transactions.
- Inventory owns where physical stock exists and how it moves.
- E-commerce and POS are sales channels using the same services.
- Product editions are created with entitlements and configuration, not separate applications.
- Inventory correctness, auditability and extensibility are mandatory across every edition.
