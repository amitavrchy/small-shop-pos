# Small Shop POS — Project Spec

> Stack: **Next.js (PWA) + NestJS + Prisma + PostgreSQL + Redis/BullMQ**  
> Lightweight, mobile-friendly POS + inventory for a single small shop (kirana / pharmacy / boutique), with barcode, stock alerts, **baki khata (store credit)**, daily reports, and offline mode.

---

## 1. Overview & Goals

Simple but not naive: one shop (optional multi-register), fast checkout on phone/tablet, credit customers ledger, and reliable offline sales sync. Advanced enough to show solid domain modeling without supermarket multi-branch complexity.

### Goals
- Sub-second barcode add on mid-range Android Chrome.
- Offline sales queue with idempotent sync.
- Baki khata: credit limit, payment allocation (FIFO on invoices), statement PDF.
- Daily Z-report / X-report.
- Stock alerts via SMS/email.

### Non-goals
- Multi-branch transfers, complex promo engine, full accounting periods (keep single cashbook + credit ledger).

---

## 2. Roles & Permissions

| Role | Permissions |
|------|-------------|
| `OWNER` | Everything: products, costs, credit limits, user manage, void, reports, settings |
| `STAFF` | Sell, receive stock (if allowed), view limited reports |
| `VIEWER` | Reports read-only |

Flags: `staffCanDiscount`, `staffCanVoid`, `staffCanSellOnCredit`, `maxDiscountPercent`.

---

## 3. Feature Modules

1. Auth (PIN login on POS + owner password on web)
2. Products & barcodes (optional variants)
3. Stock (simple qty; optional single batch/expiry for pharma mode)
4. POS cart, holds, receipts (thermal 80mm print CSS + Bluetooth print hook)
5. Payments: cash, bKash/Nagad manual confirm, card manual, **credit (baki)**
6. Customers & baki khata
7. Purchases (simple supplier bill → stock in)
8. Expenses (shop kharoch)
9. Reports: daily sales, profit estimate, credit outstanding, low stock
10. Offline sync
11. Settings: receipt header, VAT, alert thresholds

---

## 4. Advanced Business Logic

### 4.1 Baki khata (store credit)
- Customer has `creditLimitMinor`, `balanceMinor` (positive = customer owes shop).
- On credit sale: create `Sale` + `CreditLedgerEntry(type=SALE_CREDIT, debit=total)` ; balance += total; reject if `balance + total > creditLimit` unless owner override.
- On repayment: `CreditPayment` allocates to oldest open `CreditInvoice` FIFO (or manual allocation).
- Statement: running balance between dates.
- Edge: partial pay; overpay → store as customer advance (`balance` negative allowed with flag `allowAdvance`).

### 4.2 Offline sync
Same pattern as super-shop but simpler: `clientSaleId` unique; if stock goes negative on sync → `PENDING_REVIEW` and notify owner.

### 4.3 Daily reports
- **X-report**: current shift totals without closing.
- **Z-report**: close shift; lock sales in period; cash expected vs counted (`cashDifference`).
- Profit estimate: `sum(line price - product.avgCost)`; maintain weighted average cost on purchase.

### 4.4 Weighted average cost
```
newCost = (oldQty*oldCost + inQty*inCost) / (oldQty+inQty)
```
Sales do not change avg cost; adjustments can.

### 4.5 Edge cases
- Delete product with history → soft deactivate.
- Barcode relabel: keep old codes in `ProductBarcode` aliases.
- Void after Z-close → requires owner + posts reversing ledger entries.
- Concurrent credit sale vs repayment: transaction on `Customer` row `FOR UPDATE`.

---

## 5. State Machines

**Sale**: `OPEN → COMPLETED | VOID` (+ `SYNC_PENDING` / `PENDING_REVIEW`)  
**Shift**: `OPEN → CLOSED`  
**CreditInvoice**: `OPEN → PARTIAL → PAID | WRITTEN_OFF`

---

## 6. Prisma Schema Draft

```prisma
model Shop {
  id String @id @default(cuid())
  name String
  phone String?
  address String?
  vatRateBps Int @default(0)
  currency String @default("BDT")
  pharmaMode Boolean @default(false)
}
model User {
  id String @id @default(cuid())
  shopId String
  name String
  role String
  pinHash String?
  passwordHash String?
}
model Product {
  id String @id @default(cuid())
  shopId String
  name String
  sku String?
  barcode String?
  unit String @default("pcs")
  priceMinor Int
  avgCostMinor Int @default(0)
  qtyOnHand Decimal @db.Decimal(14,3)
  lowStockQty Decimal @db.Decimal(14,3) @default(0)
  expiryDate DateTime? @db.Date
  active Boolean @default(true)
  @@unique([shopId, barcode])
  @@index([shopId, name])
}
model ProductBarcode {
  id String @id @default(cuid())
  productId String
  barcode String
  @@unique([barcode])
}
model Customer {
  id String @id @default(cuid())
  shopId String
  name String
  phone String?
  creditLimitMinor Int @default(0)
  balanceMinor Int @default(0)
  @@index([shopId, phone])
}
model Sale {
  id String @id @default(cuid())
  shopId String
  registerId String?
  cashierId String
  customerId String?
  clientSaleId String @unique
  status String
  payMethod String // CASH CARD BKASH NAGAD MIXED CREDIT
  totalMinor Int
  discountMinor Int
  taxMinor Int
  creditMinor Int @default(0)
  completedAt DateTime?
  shiftId String?
  syncState String @default("SYNCED")
}
model SaleLine {
  id String @id @default(cuid())
  saleId String
  productId String
  qty Decimal @db.Decimal(14,3)
  unitPriceMinor Int
  costMinor Int
  lineTotalMinor Int
}
model CreditLedgerEntry {
  id String @id @default(cuid())
  customerId String
  type String // SALE_CREDIT PAYMENT ADJUST WRITE_OFF
  debitMinor Int @default(0)
  creditMinor Int @default(0)
  saleId String?
  paymentId String?
  note String?
  createdAt DateTime @default(now())
}
model CreditPayment {
  id String @id @default(cuid())
  customerId String
  amountMinor Int
  method String
  createdAt DateTime @default(now())
}
model CreditAllocation {
  id String @id @default(cuid())
  paymentId String
  saleId String
  amountMinor Int
}
model Purchase {
  id String @id @default(cuid())
  shopId String
  supplierName String?
  totalMinor Int
  createdAt DateTime @default(now())
}
model PurchaseLine {
  id String @id @default(cuid())
  purchaseId String
  productId String
  qty Decimal @db.Decimal(14,3)
  costMinor Int
}
model Expense {
  id String @id @default(cuid())
  shopId String
  category String
  amountMinor Int
  note String?
  spentAt DateTime @default(now())
}
model Shift {
  id String @id @default(cuid())
  shopId String
  openedBy String
  closedBy String?
  openedAt DateTime @default(now())
  closedAt DateTime?
  openingCashMinor Int
  closingCashCountedMinor Int?
  status String
}
model StockAlert {
  id String @id @default(cuid())
  productId String
  type String
  sentAt DateTime?
}
```

---

## 7. REST API

- `POST /auth/pin-login`, `POST /auth/owner-login`
- `CRUD /products`, `POST /products/barcode-lookup`
- `POST /pos/sales`, `POST /pos/sales/:id/void`
- `POST /customers`, `GET /customers/:id/statement`
- `POST /customers/:id/payments`
- `POST /purchases`
- `POST /expenses`
- `POST /shifts/open`, `POST /shifts/:id/close`
- `GET /reports/daily`, `GET /reports/z/:shiftId`
- `POST /sync/push`, `GET /sync/pull`

---

## 8. Real-time

Optional: owner dashboard listens `sale.completed`, `stock.low` via Socket.IO. Not required for MVP offline POS.

---

## 9. Jobs

- Low stock / expiry alerts (daily)
- Sync PENDING_REVIEW digest to owner
- Soft delete cleanup none (retain forever)

---

## 10. Security

- PIN lock screen after idle N minutes on POS.
- Owner-only cost visibility.
- Rate-limit PIN attempts; lockout.

---

## 11. Caching

- Product barcode map in Redis/memory per shop.
- Service worker cache for POS shell.

---

## 12. Testing

- Credit limit enforcement + concurrent repayment race.
- FIFO allocation of payments to sales.
- Weighted average cost after two purchases.
- Duplicate clientSaleId idempotency.
- Z-report cash difference math.

---

## 13. Folder Structure

```
small-shop-pos/
  Project_spec.md
  frontend/  # mobile-first PWA
  backend/
```

---

## 14. Env vars

`DATABASE_URL`, `REDIS_URL`, `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, `APP_URL`, `CORS_ORIGINS`, `SMTP_URL`, `SMS_*`, `POS_IDLE_LOCK_SECONDS`, `DEFAULT_LOW_STOCK_QTY`

---

## 15. Milestones

1. Auth + products + stock  
2. Online POS + receipts  
3. Customers + baki khata  
4. Shifts + Z-report  
5. Purchases + avg cost + profit  
6. Offline PWA sync  
7. Alerts + polish  

---

## 16. Acceptance Criteria

- [ ] Credit sale blocked when exceeding limit without owner override.
- [ ] Partial repayment reduces oldest open credit sale first.
- [ ] Offline sale syncs once; retry does not duplicate.
- [ ] Z-close prevents editing sales in that shift.
- [ ] Low stock job creates alert when qty ≤ threshold.
- [ ] Staff with `staffCanSellOnCredit=false` cannot choose CREDIT pay method.
- [ ] UI usable at 390px width for cart + pay.

---

## 17. Agent Notes

- Prefer large tap targets; scanner input focuses hidden `#scanner` field.
- Money integers (paisa). Qty Decimal.
- Seed demo shop with 50 products, 10 credit customers, open shift.
