# ART SHOP ERP (Lightweight ERP in HTML + Firebase)

## Overview
ART SHOP ERP is a lightweight, single-page ERP for a retail business selling Christmas trees and decorations. It centralizes Sales & Invoices, Inventory, and soon Purchases, Expenses, Cash/Bank, and HR. The app runs in the browser with Firebase Auth and Firestore as the backend.

## Business Context
- Sales channels: Physical store, Website, Amazon
- Sales types: Retail (primary), Wholesale (secondary)
- Payments: Cash, Card, Bank Transfer

## Current Scope (Implemented)
- Authentication (admin-gated UI)
- Sales & Invoices
  - Create/edit invoices with items, channel, payment method, description
  - Payment tracking: paymentStatus (Paid in full / Partially paid / Unpaid), paidAmount, paidAt
  - Totals: total, amountDue (derived and persisted)
  - Unit cost freezing per item (unitCost placeholder) + lineCOGS
  - View/print invoice (customer-facing layout)
  - Void invoice: marks invoice voided and restores stock
- Inventory
  - Product listing, add/edit
  - Stock updates via sales
  - Low-level stock movements: OUT for sales, IN on void

## Data Model (Firestore)
- `products`: { name, sku, quantity, createdAt }
- `invoices`: {
  customer, date, channel, paymentMethod, description,
  items[{ productId, productName, quantity, price, unitCost, lineCOGS }],
  paymentStatus, paidAmount, paidAt,
  total, amountDue,
  createdAt, createdBy, updatedAt?, updatedBy?,
  voided?, voidReason?, voidedAt?, voidedBy?
}
- `stockMovements`: { productId, quantity, type('IN'|'OUT'), source('Sale'|'Void'), refId, date, createdAt, createdBy }
- `payments`: { invoiceId, amount, method, paidAt, createdAt, createdBy, reversed?, reversedAt?, reversedBy? }
- `users`: { uid, email, role }

## Key Flows
- Create Invoice
  1) Build items, compute total
  2) Transaction: read product docs, write invoice, update product quantities (OUT)
  3) Create `stockMovements` OUT per line
  4) If a payment is entered, create `payments` doc, then recalc invoice payment state
- Edit Invoice
  - Overwrites fields and items; costs frozen per line; payment state recalculated if needed
- View/Print
  - Customer-facing details and totals; no technical info
- Void Invoice
  1) Transaction: mark invoice voided + restore product quantities
  2) Create `stockMovements` IN per line
  3) Mark related payments as `reversed=true`, then recalc payment state

## Conventions
- Roles: admin only (current UI) – extendable later (sales/warehouse/accountant/HR)
- Audit: every write includes createdBy/At; updates set updatedBy/At; void sets voidedBy/At
- Transactions: follow Firestore rule (all reads before all writes)

## Next Milestones
- Purchases: suppliers, landed cost allocation, receiving (IN movements), inventory valuation (FIFO), set item unitCost
- Cash & Bank: post receipts/payments; reconcile
- Expenses: categories, link to cash/bank
- HR: attendance, advances/deductions, monthly payroll runs

## Handover Notes
- Main entry: `index.html` (SPA) – search for “Sales Section” and “Inventory Section” comments
- Payment state recalculation helper: `recalcInvoicePaymentState(invoiceId)`
- Voiding logic: `voidInvoice(invoiceId)` (transaction + movements + payments reversal)
- Keep all stock changes mirrored in `stockMovements`
- When adding Purchases, use the same movement ledger and update item `unitCost` via FIFO
