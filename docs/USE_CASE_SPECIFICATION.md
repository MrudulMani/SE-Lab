# Use-Case Flow Specification

**Institution:** PES University, Bengaluru  
**Department:** Department of Computer Science and Engineering  
**Course:** Object-Oriented Analysis & Design / Software Engineering (Lab 1)  
**Domain:** Healthcare & Telemedicine  
**Problem Statement #15:** Pharmacy Expiry & Re-order Dispatch Engine  

---

# Core Use Case Specification: UC-01
## Dispense Medicine with Automated FEFO Allocation

| Field | Detail |
| :--- | :--- |
| **Use Case ID** | `UC-01` |
| **Use Case Name** | **Dispense Medicine with Automated FEFO Allocation** |
| **System** | Pharmacy Expiry & Re-order Dispatch Engine |
| **Primary Actor** | **Pharmacy Clerk** (Staff dispensing medicine at POS checkout terminal) |
| **Secondary Actors** | **Inventory Engine** (Background scheduler & validation daemon), **Hospital Billing System** |
| **Stakeholders & Interests** | **Pharmacy Clerk:** Desires swift, intuitive checkout without manual batch sorting.<br/>**Hospital Pharmacist / Quality Head:** Demands zero expired drug dispensation and minimal inventory loss.<br/>**Patient:** Expects potent, unexpired, and safe medication. |
| **Trigger** | The Pharmacy Clerk scans a medication barcode or enters an item SKU with the required quantity at the sales checkout counter. |

---

### 1. Preconditions
1. The Pharmacy Clerk is authenticated and logged into an active POS counter session.
2. The entered medication SKU exists in the master drug catalog with active status.
3. At least one unexpired, non-quarantined batch of the requested medication exists in the pharmacy inventory database.
4. The barcode scanner and digital checkout terminal peripheral interfaces are operational.

---

### 2. Postconditions

#### 2.1 Success End Condition (Primary Outcome)
* The requested quantity of medicine is allocated and deducted from the batch(es) having the earliest valid expiration date (First-Expired-First-Out).
* Active physical and logical stock balances are updated in real time.
* An itemized dispensing receipt/invoice detailing the medication name, allocated batch number(s), expiration date, and dosage instructions is generated.
* If stock levels breach the safety re-order threshold as a result of the sale, an automated re-order event is dispatched (`UC-06 / UC-07`).
* An immutable transaction record is logged into the system audit trail.

#### 2.2 Failure End Condition (Aborted / Rejected Outcome)
* No inventory deduction is committed, no billing transaction is processed, and stock counts remain untouched.
* The system displays an informative error notification explaining the refusal (e.g., "Insufficient Unexpired Stock", "Batch Quarantined").

---

### 3. Main Success Scenario (Basic Flow)

| Step | Actor Action | System Response |
| :---: | :--- | :--- |
| **1** | Pharmacy Clerk enters or scans the medication Item Code (SKU) and inputs the required quantity. | System validates the SKU and retrieves all active inventory batches associated with the medication. |
| **2** | — | System filters out all expired batches (Expiry Date $\le$ Current Date) and quarantined batches (`<<include>> UC-04`). |
| **3** | — | System invokes the FEFO sorting engine (`<<include>> UC-02`), ordering eligible unexpired batches chronologically by expiration date (`ASC`). |
| **4** | — | System allocates the requested quantity from the batch with the earliest expiration date having sufficient stock. |
| **5** | — | System presents the dispensing pick list on the clerk's display, showing: Medicine Name, Allocated Batch Number, Expiry Date, Rack/Shelf Location Code, Unit Price, and Total Amount. |
| **6** | Pharmacy Clerk retrieves the medication package from the indicated physical shelf and scans the batch barcode on the box. | System verifies that the scanned physical batch barcode matches the system-allocated FEFO batch. |
| **7** | Pharmacy Clerk confirms item addition to the patient's checkout cart. | System computes subtotal, applies eligible hospital discounts, and updates cart total. |
| **8** | Pharmacy Clerk finalizes checkout and processes patient payment. | System locks the allocated batch units, records the sales invoice, and generates the printed dispensing slip with batch labels. |
| **9** | — | System updates the inventory database, decrementing available stock for the allocated batch. |
| **10** | — | System evaluates post-dispensing usable stock against the SKU's safety threshold (`<<include>> UC-06`). |
| **11** | Pharmacy Clerk hands medication to patient with instructions. | System closes the active cart and returns to ready state for the next patient. |

---

### 4. Alternate Flows

#### Alternate Flow A: Multi-Batch Split Allocation under FEFO
* **Condition:** At **Step 4**, the earliest unexpired batch does not have sufficient quantity to fulfill the entire requested order, but total unexpired inventory across multiple batches is sufficient.
  * **A1.** System allocates all remaining units from the earliest unexpired batch.
  * **A2.** System calculates the remaining deficit and allocates units from the next chronologically earliest unexpired batch.
  * **A3.** System repeats the allocation sequence until the full order quantity is satisfied.
  * **A4.** System generates a multi-batch pick list on the clerk's display indicating specific unit counts and shelf locations for each respective batch.
  * **A5.** Pharmacy Clerk retrieves and scans both batches to verify physical match.
  * **A6.** Flow rejoins the **Main Success Scenario at Step 7**.

#### Alternate Flow B: Supervisor FEFO Override (`<<extend>> UC-03`)
* **Extension Point:** `[Clinical Override]` at **Step 5**.
* **Condition:** A specialized clinical prescription requires a specific manufacturer formulation batch (e.g., clinical trial protocol or patient-specific allergy restriction), bypassing the automated FEFO assignment.
  * **B1.** Pharmacy Clerk clicks "Request Clinical Batch Override" and selects the alternate valid batch.
  * **B2.** System prompts for Head Pharmacist / Supervisor administrative authorization credentials.
  * **B3.** Supervisor inputs administrative PIN and provides a mandatory override rationale code.
  * **B4.** System validates supervisor credentials and confirms the requested alternate batch is unexpired and non-quarantined.
  * **B5.** System assigns the selected batch, creates a high-priority entry in the regulatory audit log (`NFR-002`), and updates the pick display.
  * **B6.** Flow rejoins the **Main Success Scenario at Step 6**.

---

### 5. Exception Flows

#### Exception Flow E1: Stock Exhausted / Out of Stock
* **Condition:** At **Step 2**, the system finds zero unexpired units for the requested medication SKU.
  * **E1.1.** System displays alert: *"Stock Depleted: No unexpired units available for this SKU"*.
  * **E1.2.** System queries and displays therapeutic generic alternatives available in stock.
  * **E1.3.** System triggers an urgent stockout alert to the procurement engine.
  * **E1.4.** Item is removed from the active cart; Clerk notifies the patient or selects an approved substitute.

#### Exception Flow E2: Physical Batch Barcode Mismatch
* **Condition:** At **Step 6**, the physical barcode scanned by the clerk does not match the system's FEFO-allocated batch ID.
  * **E2.1.** System emits an audible warning and displays an error modal: *"Batch Mismatch! Scanned batch does not match FEFO allocation"*.
  * **E2.2.** System highlights the required shelf location and batch number again.
  * **E2.3.** The clerk returns the incorrect box and rescans the correct physical batch.
  * **E2.4.** Flow restarts from **Step 6**.

---

### 6. Special & Non-Functional Constraints
* **Latency Constraint:** The total execution time for Steps 2 through 5 (batch retrieval, expiry filtering, FEFO sort, and UI render) must be $\le 500 \text{ ms}$ (`NFR-001`).
* **Auditability:** Every dispensed transaction must record: Clerk ID, Patient ID, SKU, Batch ID, Quantity, Timestamp, and Barcode Verification Status (`NFR-002`).

---
---

# Supplementary Use Case Specification: UC-07
## Automated Purchase Order (PO) Generation & Supplier Dispatch

| Field | Detail |
| :--- | :--- |
| **Use Case ID** | `UC-07` |
| **Use Case Name** | **Dispatch Automated Purchase Order (PO)** |
| **System** | Pharmacy Expiry & Re-order Dispatch Engine |
| **Primary Actor** | **Automated Inventory Engine** (System Daemon) |
| **Secondary Actor** | **Inventory Supplier** (External Partner Procurement System) |
| **Trigger** | Available unexpired stock dips at or below the predefined safety threshold following a dispensing transaction (`UC-01`) or expiry quarantine (`UC-04`). |

### 1. Preconditions
1. Medication SKU is configured with a valid re-order point (ROP) and economic order quantity (EOQ).
2. An active supplier contract with an electronic dispatch endpoint (REST API / EDI) is registered in the vendor database.
3. No active, unfulfilled purchase order currently exists for this SKU (prevents duplicate orders).

### 2. Postconditions
* A digitally signed purchase order is generated, dispatched to the supplier's procurement endpoint, and acknowledged.
* SKU procurement status is marked as `PO_DISPATCHED`.

### 3. Main Success Scenario (Basic Flow)
1. Inventory Engine detects that total usable stock for SKU $\le$ Safety Threshold.
2. System checks the vendor catalog and retrieves primary contracted supplier details for the SKU.
3. System calculates optimal re-order quantity based on standard batch pack size and average daily consumption.
4. System generates a standardized electronic Purchase Order (PO) with a unique PO number, item SKU, quantity, price, and hospital delivery address.
5. System transmits the PO payload via encrypted TLS 1.3 REST API request to the Inventory Supplier's procurement endpoint (`NFR-002`).
6. Supplier system processes the payload, verifies order parameters, and returns an `HTTP 200 OK` acknowledgment with a Vendor Reference Number and estimated delivery date.
7. System logs receipt of acknowledgment, updates PO status to `ACKNOWLEDGED`, and alerts the Pharmacy Clerk on the dashboard.

### 4. Alternate Flow: Supplier Endpoint Timeout / Retry
* **Condition:** At Step 5, the supplier's API fails to respond within 5 seconds.
  * **A1.** System queues the PO payload in an exponential backoff retry queue (attempts at 1m, 5m, 15m).
  * **A2.** If all 3 retries fail, system sends a fallback automated encrypted email with the PO attachment to the supplier's procurement desk.
  * **A3.** System alerts the Pharmacy Clerk: *"Supplier API unreachable; fallback PO email dispatched"*.
