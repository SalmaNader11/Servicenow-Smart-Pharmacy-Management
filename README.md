# Smart Healthcare Pharmacy Management System

A scoped ServiceNow application that automates pharmacy operations end-to-end — medicine requests, stock-aware approvals, expiry monitoring, complaint handling, SLA enforcement, and a customer self-service portal.

Built as a graduation project for the **Information Technology Institute (ITI)** on ServiceNow PDI, Zurich release.

---

## Overview

| Property | Value |
|---|---|
| Platform | ServiceNow PDI |
| Release | Zurich |
| Scope | `x_1979775_pharma_0_` |
| Type | Scoped Application |

---

## The Problem

Manual pharmacy operations have predictable failure points. Stock runs out unnoticed. Medicines expire on the shelf. Customer requests get lost with no trail. Complaints go unassigned. Management has no visibility into any of it.

This system addresses each of those directly — using structured workflows, automated monitoring, role-based access, and a portal that gives customers a proper channel to interact with the pharmacy.

---

## What It Does

**Medicine Inventory**
Tracks all medicines with expiry dates, batch numbers, reorder thresholds, and supplier references. Expired medicines lock to read-only automatically. A daily scheduled job flags medicines as Expired or Near Expiry every morning without manual intervention.

**Medicine Requests**
Customers submit requests through the Service Portal. The form validates pickup date and quantity before submission and shows live stock availability via a GlideAjax call. Once submitted, an approval flow handles the rest.

**Approval Flow**
The Flow Designer flow routes every request to the Pharmacy Admin for review. On approval, it checks available stock — if sufficient, it deducts the quantity and notifies the customer their medicine is ready. If stock is insufficient, it rejects automatically and notifies the customer. On admin rejection, the request closes with a rejection notification.

**Low Stock Alerts**
When a medicine's quantity drops to or below its reorder level, a Business Rule fires a custom event that triggers notifications to both the Admin and the supplier's contact email.

**Complaint Management**
Customers report complaints through the portal via a Record Producer. Every complaint is auto-assigned to the Pharmacy Admin group, classified by type, and tracked through resolution. Staff cannot close a complaint without filling in Resolution Notes.

**SLA Enforcement**
Requests must be responded to within 1 hour. Complaints must be resolved within 2 hours. Warning and breach notifications fire automatically at 75% and 100% of each target.

**Service Portal**
The customer-facing portal at `/pharma` gives customers everything they need — submit requests, track their history, and report complaints — without ever accessing the ServiceNow backend.

**Dashboard**
A live Admin dashboard shows inventory health, request backlog by stage, top requested medicines, and low stock medicines at a glance.

---

## Data Model

| Table | Type | Purpose |
|---|---|---|
| `pharmacy_medicine` | Custom | Medicine inventory |
| `supplier` | Custom | Supplier details |
| `sc_req_item` | Native | Customer medicine requests |
| `incident` | Native | Customer complaints |

---

## Roles

| Role | Access |
|---|---|
| Admin | Full access — approvals, all data, dashboard |
| Pharmacist | Inventory read/write, no delete |
| Customer | Portal only — own requests and complaints |

---

## Tech Stack

ServiceNow PDI · Flow Designer · Business Rules · Client Scripts · Script Includes · GlideAjax · Scheduled Jobs · Service Portal · AngularJS · ACLs · SLA Engine · Reports & Dashboard

---

## Repository Contents

| Path | Contents |
|---|---|
| `/docs` | Full project documentation — data model, implementation, test cases, results |
| `/presentation` | Project presentation slides |
| `/update-set` | Exportable ServiceNow update set XML — importable to any PDI |
| `/screenshots` | Screenshots of all major features |

> For full technical detail — field definitions, script bodies, ACL rules, test cases, and screenshots — see the documentation in `/docs`.

---

## Author

**Salma Nader**  
ITI Graduation Project · ServiceNow Administration & Development
