# Requirements Engineering Specification

**Institution:** PES University, Bengaluru  
**Department:** Department of Computer Science and Engineering  
**Course:** Object-Oriented Analysis & Design / Software Engineering (Lab 1)  
**Domain:** Healthcare & Telemedicine  
**Problem Statement #15:** Pharmacy Expiry & Re-order Dispatch Engine  

---

## 1. Problem Context & Overview

Hospital pharmacies operate in high-stakes environments where dispensing precision, regulatory compliance, and stock continuity are critical to patient outcomes. Traditional manual inventory checks often lead to:
1. **Medication Wastage & Expiry Spoilage:** Newer stock batches are inadvertently sold ahead of older stock, causing earlier batches to cross expiration dates unnoticed.
2. **Patient Safety Risks:** Risk of dispensing near-expired or expired medication due to human oversight during peak sales checkout hours.
3. **Critical Stockouts:** Manual stock count delays result in sudden depletion of life-saving emergency medicines before replenishment purchase orders can be dispatched.

The **Pharmacy Expiry & Re-order Dispatch Engine** is an automated stock management and procurement system designed to solve these challenges. It tracks medicine batch expiration dates in real time, enforces First-Expired-First-Out (**FEFO**) dispensing rules during patient checkout, isolates expired inventory, monitors safety stock thresholds, and dispatches automated electronic purchase orders (PO) to inventory suppliers.

### Stakeholders & Target Actors
* **Pharmacy Clerk (Primary User):** Operates checkout terminals, scans prescriptions, retrieves FEFO-allocated stock from shelves, and inwards newly received batches from delivery suppliers.
* **Inventory Supplier (External Stakeholder / Partner):** Receives automated electronic purchase orders (EDI/REST API), confirms order fulfillment, and delivers scheduled pharmaceutical consignments.
* **Hospital Pharmacy Administrator / Auditor:** Audits batch tracking logs, reviews inventory health, and configures safety stock thresholds and supplier contracts.

---

## 2. Requirements Table (Summary Matrix)

| Requirement ID | Type | Priority | Description Summary |
| :--- | :--- | :--- | :--- |
| **FR-001** | Functional (Dispensing) | **High** | Enforce automated FEFO batch allocation during sales checkout. |
| **FR-002** | Functional (Quality Assurance) | **High** | Batch expiry tracking, near-expiry alerts, and automated expired stock quarantining. |
| **FR-003** | Functional (Inventory Control) | **High** | Continuous safety stock monitoring and automated re-order breach detection. |
| **FR-004** | Functional (Procurement Integration) | **Medium** | Automated electronic purchase order (PO) generation and supplier dispatch. |
| **FR-005** | Functional (Inwarding & Reconciliation) | **High** | Physical goods receipt verification, batch recording, and PO reconciliation. |
| **NFR-001** | Non-Functional (Performance & Scalability) | **High** | FEFO query latency $\le$ 500 ms at 100 concurrent terminals; PO dispatch latency $\le$ 3 s. |
| **NFR-002** | Non-Functional (Security & Auditability) | **High** | Role-Based Access Control (RBAC), AES-256/TLS 1.3 encryption, and immutable audit logs. |

---

## 3. Detailed Functional Requirements (FR-001 to FR-005)

### Requirement: FR-001
* **ID:** `FR-001`
* **Type:** Functional (Stock Dispensing Logic)
* **Priority:** `High`
* **Description:**  
  The system shall enforce First-Expired-First-Out (FEFO) picking order by automatically prioritizing and allocating medicine batches with the nearest valid expiry date during sales checkout and prescription fulfillment.
* **Acceptance Criteria:**  
  * **Pass:** When a medication SKU is entered for checkout, the system automatically selects units from the earliest unexpired batch. If the requested quantity exceeds the available stock of the earliest batch, the remaining balance is sequentially allocated from the next earliest unexpired batch.
  * **Fail:** The system assigns or recommends stock units from a newer batch while an older unexpired batch of identical SKU with active stock is available in inventory.
* **Rationale:**  
  Eliminates stock aging on pharmacy shelves, minimizes clinical waste and write-offs, and ensures older compliant stock is consumed prior to newer deliveries.

---

### Requirement: FR-002
* **ID:** `FR-002`
* **Type:** Functional (Quality Assurance & Expiry Control)
* **Priority:** `High`
* **Description:**  
  The system shall continuously track the expiration dates of all inventory batches, trigger proactive alerts for batches approaching expiry within a configurable buffer (e.g., $\le$ 30 days), and automatically lock any expired batch into a quarantined status preventing checkout.
* **Acceptance Criteria:**  
  * **Pass:** All batches within the 30-day expiry window are highlighted in yellow on the clerk dashboard. At 00:00:00 on the date of expiration, the system automatically flips the batch status to `QUARANTINED`, instantly disabling it from sales selection and notifying the supervisor.
  * **Fail:** A batch past its expiration date remains selectable, billable, or dispensive at POS terminals, or batches within 30 days of expiry fail to trigger warning indicators.
* **Rationale:**  
  Dispensing expired pharmaceuticals introduces fatal clinical risks, severe medical malpractice liability, and regulatory sanctions under drug enforcement laws.

---

### Requirement: FR-003
* **ID:** `FR-003`
* **Type:** Functional (Inventory Monitoring & Threshold Triggers)
* **Priority:** `High`
* **Description:**  
  The system shall automatically monitor aggregate usable (unexpired) inventory levels against predefined safety re-order thresholds for each drug SKU and immediately trigger a purchase order request when available stock breaches the threshold.
* **Acceptance Criteria:**  
  * **Pass:** Following any stock deduction (dispensing, damage adjustment, or expiry quarantine), if usable inventory $\le$ safety threshold, the system flags the SKU as `REORDER_TRIGGERED` and dispatches an event to the procurement engine within 5 seconds.
  * **Fail:** Usable inventory drops at or below the safety threshold without triggering a purchase order event, or the system triggers re-orders when stock exceeds the threshold.
* **Rationale:**  
  Prevents stockouts of essential and emergency medications, guaranteeing continuous clinical availability without relying on manual periodic stock audits.

---

### Requirement: FR-004
* **ID:** `FR-004`
* **Type:** Functional (Procurement & Supplier Electronic Dispatch)
* **Priority:** `Medium`
* **Description:**  
  The system shall compile triggered re-order events into a standardized electronic Purchase Order (PO) document—specifying item SKU, drug generic name, required quantity, packaging specifications, delivery destination, and delivery due date—and transmit it directly to the contracted Inventory Supplier via automated electronic interface (EDI/REST API/Secure Email).
* **Acceptance Criteria:**  
  * **Pass:** An electronic PO with a globally unique PO-ID and complete metadata is generated and transmitted to the supplier's endpoint, receiving an automated HTTP 200 / ACK response and updating internal status to `PO_DISPATCHED`.
  * **Fail:** The system dispatches a PO with missing mandatory procurement fields, transmits to an unauthorized supplier endpoint, or fails to log transmission failures with retry attempts.
* **Rationale:**  
  Eliminates manual purchase order drafting delays, prevents data entry errors, shortens replenishment lead times, and maintains a transparent supplier order pipeline.

---

### Requirement: FR-005
* **ID:** `FR-005`
* **Type:** Functional (Goods Receipt, Batch Inwarding & Reconciliation)
* **Priority:** `High`
* **Description:**  
  The system shall enable the Pharmacy Clerk to log incoming stock consignments delivered by the Inventory Supplier, verifying batch numbers, manufacturing dates, expiry dates, and received quantities against the corresponding Purchase Order before activating the stock in live inventory.
* **Acceptance Criteria:**  
  * **Pass:** When the clerk scans/enters shipment details matching an open PO, the system creates unique batch ledger entries, increments active stock counts, marks the PO as `FULFILLED`, and logs the receiving clerk's credentials.
  * **Fail:** Stock is added to active inventory without mandatory batch number or expiration date inputs, or the system closes a PO where delivered counts deviate from order counts without recording a discrepancy log.
* **Rationale:**  
  Ensures rigorous traceability from manufacturing batch to patient dispensation, verifies vendor compliance with delivery contracts, and prevents counterfeit or unregistered drugs from entering circulation.

---

## 4. Detailed Non-Functional Requirements (NFR-001 & NFR-002)

### Requirement: NFR-001
* **ID:** `NFR-001`
* **Type:** Performance, Latency & Scalability
* **Priority:** `High`
* **Description:**  
  The system shall maintain low latency and high transaction throughput under peak hospital checkout loads, executing FEFO batch allocation calculations in sub-second time and dispatching automated purchase orders with minimal latency.
* **Acceptance Criteria:**  
  * **Pass:** Under a simulated peak workload of 100 concurrent checkout terminals, the 95th percentile ($P_{95}$) response time for FEFO batch calculation and checkout allocation is $\le 500 \text{ ms}$; the latency between a threshold breach event and supplier PO dispatch is $\le 3.0 \text{ seconds}$.
  * **Fail:** FEFO allocation query latency exceeds $1.0 \text{ second}$, or batch re-order dispatch events are dropped or queued for more than $10 \text{ seconds}$ during concurrent peak activity.
* **Rationale:**  
  Hospital outpatient pharmacies experience concentrated rush hours; delays in batch calculation stall checkout counters, prolong patient wait times, and degrade emergency care responsiveness.

---

### Requirement: NFR-002
* **ID:** `NFR-002`
* **Type:** Security, Integrity & Regulatory Compliance
* **Priority:** `High`
* **Description:**  
  The system shall enforce Role-Based Access Control (RBAC) across all inventory operations, encrypt all inventory and supplier communications both at rest (AES-256) and in transit (TLS 1.3), and maintain an immutable, tamper-evident audit log of every batch state change, FEFO override, and purchase order transaction.
* **Acceptance Criteria:**  
  * **Pass:** Unauthenticated or unauthorized attempts to override FEFO allocations, edit batch expiration dates, or dispatch POs are blocked with `HTTP 403 Forbidden`; all database records are verified to use AES-256 encryption; all transaction audit logs contain user ID, workstation ID, timestamp (UTC), action type, and cryptographic hash verifying non-repudiation.
  * **Fail:** Any non-administrative user can alter batch expiration dates, audit log entries can be modified or deleted, or unencrypted supplier communications occur.
* **Rationale:**  
  Pharmaceutical inventory systems are subject to strict legal and clinical regulations (FDA 21 CFR Part 11, HIPAA, and CDSCO). Any unauthorized tampering with batch records or expiration dates can lead to catastrophic medical harm, fraud, and severe legal liability.
