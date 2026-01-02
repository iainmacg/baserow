# Baserow as an R&D MES: Architectural Overview

## Executive Summary
Baserow is highly suitable as the **Data Layer** and **User Interface** for an R&D-scale MES (Manufacturing Execution System) due to its flexibility, which is crucial for high-mix/low-volume environments. However, it lacks native "Transactional Logic" (state machines) required to rigidly enforce process flows.

**Verdict:** Suitable, provided you implement a "Logic Layer" (via Plugins or API Wrappers) to handle Lot Moves and History logging.

---

## 1. Suitability Analysis for R&D Semiconductor Mfg

### Strengths (The "Flexibility" Aspect)
*   **Rapid Schema Evolution:** in R&D, you frequently add new metrology fields or experimental parameters. Baserow allows Engineers to add these columns instantly without database migrations.
*   **Relational Integrity:** The "Link to Table" feature perfectly models the complex relationships between `Lots`, `Flows`, and `Equipment`.
*   **API-First:** Every table and row is immediately accessible via API, making integration with automated test equipment (ATE) or metrology tools straightforward.

### Weaknesses (The "Control" Aspect)
*   **Lack of State Machine:** Baserow allows users to edit a "Status" field directly. In an MES, you want to *prevent* a user from changing "Etch" to "Deposition" unless the "Etch" step is actually completed.
*   **Audit Trail:** While Baserow has basic row history, an MES requires a dedicated, queryable `History` table (Who moved Lot X from Step A to Step B at Time T?).
    *   *Note on Versions:* The Enterprise version includes a persistent **Audit Log** (`AuditLogEntry` table). The Free/Open-Source version has an `Action` table used for Undo/Redo, but it is **temporary** (entries are deleted after ~2 hours by default) and cannot be used for compliance or long-term tracking.

---

## 2. Proposed Data Model
To balance Flexibility and Control, we recommend the following relational schema:

### Core Tables
1.  **`Process_Steps`**: A library of all possible manufacturing steps (e.g., "Gate Oxide", "Litho L1").
    *   *Fields:* Name, Description, Work Instructions (Rich Text), Required Equipment (Link).
2.  **`Process_Flows`**: Defines the "Recipe".
    *   *Fields:* Name, Version.
    *   *Structure:* This is the header. The actual steps are defined in a child table or a "Many-to-Many" link table (e.g., `Flow_Steps`) with an "Order" field.
3.  **`Equipment`**: Tools available in the fab.
    *   *Fields:* Tool ID, Status (Up/Down), Qualification Date.
4.  **`Lots`** (The WIP): The physical wafers.
    *   *Fields:* Lot ID, **Current_Flow** (Link), **Current_Step** (Link), Status (Waiting, In-Process, Hold), Priority.
5.  **`Transactions`** (The History): **Crucial for Traceability.**
    *   *Fields:* Lot (Link), From_Step, To_Step, User, Timestamp, Comment.
    *   *Note:* This table should be Read-Only for most users.

---

## 3. Gap Analysis & Solutions

### Gap 1: The "Move" Operation (Logic)
**The Problem:** An operator should not manually change the `Current_Step` dropdown. They should click a "Move" button that validates the move and automatically updates the step.
**The Solution:**
*   **Option A (Low Code):** Use Baserow's **Application Builder**. Create a "Move" button that triggers a Workflow Action.
    *   *Constraint:* You need a Custom Workflow Action (Plugin) to perform the "Transaction" (Update Lot + Create History Record) atomically.
*   **Option B (External Logic):** A simple Python/Node script running as a webhook listener.
    *   *Flow:* User clicks button -> Webhook -> Python Script validates logic -> Script updates Baserow via API.

### Gap 2: Permissions (Read/Write Control)
**The Problem:** Operators should see work instructions but only be able to "Move" lots, not delete them or change engineering specs.
**The Solution:**
*   **Role-Based Access:** Use Baserow's workspace permissions.
    *   *Engineers:* `Admin` or `Member` (Can edit Flow definitions).
    *   *Operators:* Limited `Member` role.
*   **Interface Level Security:** Build a dedicated **Baserow App** (frontend) for Operators.
    *   The App displays the "Lot List" and "Move Button".
    *   The underlying Database tables are kept hidden or read-only in the raw view.

---

## 4. Handling R&D Specifics

### Split & Merge (Genealogy)
R&D often splits a lot of 25 wafers into two sub-lots of 12 and 13 to test different conditions.
*   **Implementation:** Add a `Parent_Lot` (Link to `Lots`) field in the `Lots` table.
*   **Visualization:** Use a custom Plugin or a recursive API query to build a "Genealogy Tree" visualization in the Dashboard.

### Ad-Hoc Flow Changes
Engineers often need to insert a "Rework" step for a specific lot without changing the master Flow.
*   **Strategy:** Add a `Custom_Next_Step` (Link to `Process_Steps`) field on the `Lot` table.
*   **Logic:** The "Move" logic checks: *Is `Custom_Next_Step` set? If yes, go there. If no, follow the standard `Process_Flow`.*

---

## Recommendations
1.  **Start with the Data Model:** Build the tables described above.
2.  **Build a "Move" Plugin:** This is the highest leverage engineering task. A simple plugin that exposes a `move_lot(lot_id, target_step)` action will transform Baserow from a database into a functional MES.
3.  **Use the App Builder:** Don't let operators edit the database grid directly. Give them a simplified view.
