# Smart Healthcare Pharmacy Management System

A scoped ServiceNow application that automates pharmacy operations end-to-end — from customer medicine requests and stock-aware approvals to expiry monitoring, complaint resolution, SLA enforcement, and a branded self-service portal.

Built as a graduation project for the **Information Technology Institute (ITI)** on ServiceNow PDI (Zurich release).

---

## Overview

| Property | Value |
|---|---|
| Platform | ServiceNow Personal Developer Instance (PDI) |
| Release | Zurich |
| Application Scope | `x_1979775_pharma_0_` |
| Application Type | Scoped Application |
| Track | ServiceNow Administration & Development |

---

## Problem Statement

Manual pharmacy operations create risks that compound over time:

- Stock levels are tracked on paper — shortages go undetected until a customer asks for something that isn't there
- Expired medicines stay on shelves because no one is systematically checking dates
- Customer requests are handled informally with no approval trail, no SLA, and no accountability
- Complaints go unassigned and unresolved with no structured routing or follow-up
- Management has no real-time visibility into inventory or request volumes

This project addresses every one of these problems using native ServiceNow capabilities — structured workflows, automated monitoring, role-based access, and a customer-facing portal.

---

## Data Model

Two custom tables and two reused native tables.

| # | Table | Type | Purpose |
|---|---|---|---|
| 1 | `x_1979775_pharma_0_pharmacy_medicine` | Custom | Medicine inventory — stock, expiry, batch, reorder |
| 2 | `x_1979775_pharma_0_supplier` | Custom | Supplier details — referenced from medicine inventory |
| 3 | `sc_req_item` | Native | Customer medicine requests (RITMs) via Service Catalog |
| 4 | `incident` | Native | Customer complaints via custom pharmacy_complaint view |

**Relationships:**
- `pharmacy_medicine` → `supplier` via Reference field
- `sc_req_item` → `pharmacy_medicine` via catalog variable (medicine_name)
- `incident` → Pharmacy Admin group via auto-assignment Business Rule
- `sys_user` → custom roles (admin, pharmacist, customer)

---

## Roles & Access Control

| Role | Access | Responsibilities |
|---|---|---|
| `admin` | Full | Approve/reject requests, manage all data, view dashboard |
| `pharmacist` | Inventory-focused | Read/write inventory and suppliers, no delete |
| `customer` | Portal only | Submit requests, track status, report complaints |

ACLs are enforced at table level. Data Policies enforce `medicine_name` and `quantity` as mandatory at database level — not just UI level.

---

## Key Features

### Medicine Inventory
- 11 fields across 3 form sections — Basic Info, Inventory Details, Supplier
- UI Policy: status = Expired → all fields lock to read-only automatically
- 15 sample records across all status types: Active, Low Stock, Near Expiry, Expired

### Service Catalog — Medicine Request
- 5 catalog variables including a Reference field to the inventory table
- 3 client scripts: pickup date validation, quantity validation, live availability check via GlideAjax
- Script Include: `AvailableQuantityCheck` — server-side, client-callable

### Approval Flow (Flow Designer)
- Trigger: RITM created from Medicine Request catalog item
- Sets stage to Under Review → notifies Admin
- Admin approves or rejects
- On approval: checks available stock → if sufficient, deducts quantity and sets stage to Ready for Pickup → sends PickupReady email to customer
- If stock is insufficient: automatically rejects with notification to customer
- On rejection: sets stage to Rejected → sends rejection email
- Staff clicks Mark as Completed UI Action when customer collects medicine

### Stock & Expiry Monitoring
- **LowStockAlert** Business Rule: fires `pharmacy.low_stock` event when quantity ≤ reorder level → notifies Admin and Supplier
- **ExpiryCheck** Scheduled Job: runs daily at 06:00 — scans all medicines, sets status to Expired or Near Expiry based on date
- **BlockExpiredMedicine** Business Rule: blocks approval if the requested medicine has expired between submission and approval

### Complaint Management
- Custom view `pharmacy_complaint` on native Incident table
- Custom field `u_choice_1` — complaint type: Wrong Medicine, Delay in Service, Other
- **ComplaintAutoAssign** Business Rule: auto-assigns every new complaint to Pharmacy Admin group
- UI Policy: Resolution Notes become visible and mandatory when state = Resolved or Closed
- Record Producer: `ReportProblem` — customers submit complaints through the portal

### SLA Management
- **MedicineRequestSLA**: 1-hour response target — starts at Under Review, stops at Approved or Rejected
- **ComplaintSLA**: 2-hour resolution target — starts at New, stops at Resolved
- Warning notifications at 75%, breach notifications at 100%

### Notifications (10 total)

| Notification | Trigger | Recipient |
|---|---|---|
| RequestReceived | RITM created | Customer |
| NewRequestAdmin | RITM created | Pharmacy Admin group |
| PickupReady | Flow — approved branch | Customer |
| RequestRejected | Flow — rejected / insufficient stock | Customer |
| LowStockNotifAdmin | pharmacy.low_stock event | Admin group |
| LowStockNotifSupplier | pharmacy.low_stock event | Supplier contact email |
| SLA Warning — Request | MedicineRequestSLA at 75% | Admin group |
| SLA Breach — Request | MedicineRequestSLA at 100% | Admin group |
| SLA Warning — Complaint | ComplaintSLA at 75% | Admin group |
| SLA Breach — Complaint | ComplaintSLA at 100% | Admin group |

All notifications use a consistent HTML email template with a navy header.

### Service Portal — /pharma
- Built on Jolla theme with custom navy and teal color scheme
- **Welcome widget**: dynamic first name greeting using AngularJS and `gs.getUser().getFirstName()`
- Quick Links: My Requests and Report a Problem
- Medicine Request catalog item accessible directly from the portal
- Customers see only their own requests — no backend access

### Dashboard & Reports
- **Inventory by Status** — pie chart showing Active, Low Stock, Near Expiry, Expired counts
- **Low Stock Medicines** — filtered list of medicines needing reorder
- **Requests by Stage** — bar chart showing request backlog per stage
- **Top Medicines** — most frequently requested medicines
- All pinned to a single Admin-only dashboard

---

## End-to-End Flow

```
Customer submits Medicine Request on /pharma portal
        ↓
RITM created → stage = Under Review → Admin notified → SLA starts
        ↓
Admin reviews and approves or rejects
        ↓ (approved)
Flow checks available stock
        ↓ (sufficient)                    ↓ (insufficient)
Stock deducted                        Stage = Rejected
Stage = Ready for Pickup              Customer notified
PickupReady email sent
        ↓
Staff clicks Mark as Completed
Stage = Complete → record closes
```

---

## Implementation Phases

| Phase | What was built |
|---|---|
| P1 | Medicine inventory table, supplier table, ACLs, Data Policies, UI Policies, sample data |
| P2 | Medicine Request catalog item, variables, stage set, client scripts, Script Include |
| P3 | Flow Designer — Medicine Approval Flow with stock validation and deduction |
| P4 | Low stock Business Rule, custom event, Admin and Supplier notifications |
| P5 | Expiry monitoring scheduled job, BlockExpiredMedicine Business Rule |
| P6 | SLA definitions, warning and breach notifications |
| P7 | Incident complaint view, complaint type field, auto-assignment BR, Record Producer |
| P8 | Full notification system — 10 notifications with HTML templates |
| P9 | Reports and Admin dashboard |
| P10 | Service Portal — theme, widgets, home page, quick links |

---

## Challenges

- **Business Rule not firing**: UI Action was using `new GlideRecord()` instead of `current` — switching to `current.update()` triggered the BR correctly
- **GlideAjax async**: Availability check was returning undefined because the form field updated before the server responded — fixed by moving the update inside the callback
- **SLA conditions**: SLA wasn't starting because conditions matched display values instead of stored integer field values
- **Portal CSS overrides**: ServiceNow's Bootstrap layers override widget CSS — fixed by scoping styles to container classes

---

## Repository Structure

```
smart-pharmacy-servicenow/
├── README.md
├── docs/
│   └── Smart_Pharmacy_Documentation.docx
├── presentation/
│   └── Smart_Pharmacy_Presentation.pptx
├── update-set/
│   └── smart_pharmacy_update_set.xml
├── screenshots/
│   └── portal_home.png
│   └── approval_flow.png
│   └── inventory_table.png
│   └── dashboard.png
```

---

## Tech Stack

- **Platform**: ServiceNow PDI (Zurich)
- **Development**: Scoped Application, Flow Designer, Business Rules, Client Scripts, Script Includes, GlideAjax, Scheduled Jobs
- **Frontend**: ServiceNow Service Portal, AngularJS widgets
- **Reporting**: ServiceNow Reports & Dashboard
- **Security**: ACLs, Data Policies, Role-based access

---

## Future Enhancements

- Automated reorder: generate purchase order and notify supplier when stock hits reorder level
- Prescription verification via OCR on the uploaded prescription PDF
- Multi-branch inventory support with consolidated reporting
- Mobile push notifications for request status updates
- Performance Analytics for time-series inventory and request trend analysis

---

## Author

**Salma Nader**
ITI Graduation Project — ServiceNow Administration & Development Track
