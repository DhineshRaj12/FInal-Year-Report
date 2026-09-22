# AR Events — Real-Time Tent & Equipment Allocation Framework

**UNITAR International University — ITWB2914 Minor Project**
Faculty of Artificial Intelligence and Frontier Technologies (FAIFT)

> Enhancing Event Logistics Efficiency Through an Integrated Real-Time Tent and Equipment Allocation Framework

[![Status](https://img.shields.io/badge/status-analysis%20%26%20design%20phase-blue)]()
[![Programme](https://img.shields.io/badge/programme-BIT%20(Data%20Analytics)-informational)]()
[![University](https://img.shields.io/badge/university-UNITAR-red)]()

---

## 📖 Overview

**AR Events** is a medium-sized wedding and feasting equipment rental business in Bandar Sunway, Petaling Jaya, managing **1,000+ rentable assets** (canopies, tables, chairs, cooling systems, and decor) with just four employees. Today, every booking, allocation, invoice, and payment lives on the owner's mobile phone — there is no single view of what's committed on a given day, no way to measure the profit of an event, and no visibility into seasonal demand.

This project designs an **integrated real-time allocation framework** to solve that: a system that tracks equipment availability across overlapping booking windows, captures cost/labour/revenue data at the individual event level, and gives the owner a dashboard for profit and demand trends.

This repository is a **minor project deliverable** — it covers the **analysis and design phases only**. Implementation, testing, and deployment are scoped for a follow-on major project.

---

## 📁 Repository Structure

```
.
├── Documentation/
│   └── Final_Year_Report.pdf      # Full project report (99 pages)
├── Figma/
│   └── AR-Events-Interface-Mockups.fig   # Interface mockups (or export)
└── README.md
```

- **`Documentation/`** — the complete project paper report: literature review, methodology, requirements, system analysis, system design, and conclusion.
- **`Figma/`** — the UI/UX mockups for the nine owner-facing screens referenced in the report (Figure 7–15).
- **`README.md`** — this file.

🔗 **Figma design file:** [AR Events Allocation System — Interface Mockups](https://www.figma.com/design/yUFhobt6ZNW9iqVnWbtQv4/AR-Events-Allocation-System-%E2%80%94-Interface-Mockups?node-id=2-26)

---

## 🎯 Project Objectives

| # | Objective |
|---|-----------|
| 1 | Understand AR Events' current booking, allocation, labour, and record-keeping practices through direct engagement with the business owner. |
| 2 | Develop a real-time allocation model that tracks asset-level availability across overlapping occupancy windows and surfaces uncommitted quantity per equipment variant. |
| 3 | Design a data structure capturing cost, labour, and revenue at the individual event level to enable per-event, per-event-type, and per-asset-category profit analysis. |
| 4 | Develop an analytical approach for identifying seasonal demand patterns, plus a dashboard giving the owner visibility into bookings and profit (daily/weekly/monthly). |

---

## 🧩 Problem Being Solved

- A rental stock of **1,000+ items** is coordinated entirely from a **mobile phone list**.
- Bookings, invoices, payments, and transport records are kept **separately**, with no single source of truth.
- The owner **cannot calculate profit per event** — even though all the data needed to do so is already generated and then discarded.
- Equipment goes out **one day before** an event and returns **one day after**, meaning every booking occupies a **3-day window**, not a single date — so two events on different dates can still compete for the same stock.

---

## ✅ Scope

**In scope (this minor project):**
- Requirements gathering from AR Events' owner (sole intended user)
- Documentation of the current booking/allocation/recovery/labour process
- Functional & non-functional requirements specification
- Hardware, software, network, and data resource specifications
- Use case diagram and data flow diagrams (context + level 1)
- Technical, economic, operational, schedule, and legal feasibility study
- Entity-relationship database design
- System architecture and interface design (owner-facing screens)
- Analytical design for demand trends and profit reporting

**Out of scope (deferred to the major project):**
- Coding, implementation, and deployment
- Unit, integration, and user acceptance testing
- Third-party payment/accounting/logistics integration
- Worker-facing interfaces or payroll processing
- Direct customer-facing booking functionality

---

## 👤 Target User

The system is designed for a **single, non-technical user**: **Mr Appala Raj**, proprietor of AR Events, who currently runs the business from his smartphone. The four workers who handle transport and assembly are **not system users**, though their labour, equipment handling, and wages are recorded as data.

---

## 🛠️ System Requirements Summary

### Functional Requirements (excerpt — 20 total, see report §4.3)
- Maintain a register of equipment variants (size, material, colour) with owned quantities
- Record bookings with occupancy windows derived automatically from the event date (overridable)
- Calculate uncommitted quantity per variant across a requested window and reject over-commitment
- Record dispatch/return, discrepancies, loss/damage charges, and event costs
- Track worker wages per booking, per worker, with settlement status
- Calculate per-event profit, generate invoices, and report profitability by event type/category
- Offer substitute variants when a requested size is unavailable, logging the outcome
- Generate a demand index per equipment category and a profit dashboard

### Non-Functional Requirements (excerpt — 9 total, see report §4.4)
- Booking entry completable in ~60 seconds
- Operable on a smartphone browser, without formal training
- Availability queries return within 3 seconds
- Tolerant of intermittent mobile connectivity
- Recoverable following device loss (remote hosting + backups)
- Restricted to the authenticated owner

---

## 🏗️ System Architecture

A **three-tier architecture**:

1. **Presentation tier** — responsive, browser-based web app (no native app/install required)
2. **Application tier** — server-side logic for bookings, availability calculation, quoting, invoicing; scheduled (not real-time) analytics jobs
3. **Data tier** — relational database management system, chosen for referential integrity across bookings, allocations, labour, and cost records

The client tolerates intermittent connectivity by holding bookings in local state until submission succeeds.

---

## 🗄️ Database Design

**12 entities**, including:

`CUSTOMER` · `BOOKING` · `EQUIPMENT_CATEGORY` · `EQUIPMENT_VARIANT` · `ALLOCATION` · `DISCREPANCY` · `EVENT_COST` · `WORKER` · `LABOUR_LINE` · `PAYMENT` · `SUBSTITUTION_LOG` · `RATE_CARD`

Key design decisions:
- **Category vs. variant separation** — colour and size are allocatable dimensions (e.g. maroon vs. gold tablecloths are separate availability pools), not descriptive attributes.
- **Per-worker labour tracking** — wages are recorded per worker per booking (`LABOUR_LINE`), since settlement status differs per person, not per job.
- **Substitution outcomes are logged**, not just offered — enabling sub-rental costs to be traced back to the booking that caused them.

Full ER diagram and data dictionary: see `Documentation/Final_Year_Report.pdf`, §5.3.

---

## ⚙️ Allocation Logic (Core Algorithm)

Availability is resolved as an **interval-overlap problem**, not a date-match problem, because every booking occupies a 3-day window:

```
FUNCTION available_quantity(variant_id, window_start, window_end):
    owned = EQUIPMENT_VARIANT.owned_quantity WHERE variant_id matches
    peak_committed = 0

    FOR each day d FROM window_start TO window_end:
        committed_on_d = SUM(ALLOCATION.quantity_committed)
            WHERE ALLOCATION.variant_id matches
            AND ALLOCATION.booking.status IN (confirmed, dispatched)
            AND d BETWEEN booking.window_start AND booking.window_end

        IF committed_on_d > peak_committed:
            peak_committed = committed_on_d

    RETURN owned - peak_committed
```

The **peak** (not average) committed quantity is used — a variant is unavailable if it's fully committed on even one day within the requested window. Cancelled and enquiry-status bookings are excluded so unconfirmed leads don't block real availability.

---

## 📊 Interface Screens

Nine owner-facing screens were designed with a **60-second interaction budget** in mind (see `Figma/` and report §5.6):

| Figure | Screen |
|--------|--------|
| 7 | Dashboard |
| 8 | Availability Check |
| 9 | New Booking |
| 10 | Record Return |
| 11 | Seasonal Demand |
| 12 | Add Equipment Item |
| 13 | Event Costs |
| 14 | Labour Record |
| 15 | Add Labour |

---

## 📚 Methodology

- **Model:** Waterfall (with rationale and limitations documented in the report)
- **Requirements gathering:** structured interview with the business owner (12-area guide + 8-area follow-up)
- **Literature review:** 15 sources (2020–2026) covering SME inventory management, seasonal demand forecasting under data scarcity, BI adoption, asset tracking, and a comparative analysis of six commercial rental management platforms

---

## 🔍 Key Findings

The project's central finding isn't a design decision — it's operational: **AR Events cannot currently state the profit made from any single event**, even though every element needed to calculate it (labour cost, revenue, loss recovery) is already generated by the business. What's missing isn't data — it's **structure**.

### Known Limitations
- **Transport cost is estimated**, not metered — the business's own vehicles have no per-trip invoices
- **Pricing structure is an open design item** — whether pricing is a per-unit rate summation or a tier-based package model was not confirmed before the report was finalized
- **Demand forecasting is specified but not demonstrated** — only one season of invoice data was available at time of writing; the method requires at least two full seasonal cycles

---

## 🎓 Author

**Dhinesh Raj A/L Philip Jason**
Matric No: MC231025222
Bachelor of Information Technology (Data Analytics), UNITAR International University
Supervisor: Assoc. Prof. Dr. Fakhrul Hazman Yusoff

---

## 📄 License

This repository contains academic coursework submitted in partial fulfilment of the requirements for the Bachelor of Information Technology degree at UNITAR International University. Shared for portfolio and educational purposes.
