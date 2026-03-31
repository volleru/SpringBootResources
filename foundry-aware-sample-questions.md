# Palantir Foundry Aware — Sample Questions & Answers

> **Exam format:** ~60 questions, 120 minutes, 70% passing score, free via [learn.palantir.com/foundry-aip-aware](https://learn.palantir.com/foundry-aip-aware)

---

## Foundry Platform & Architecture

**Q1. What is the primary purpose of the Palantir Ontology in Foundry?**

> **A:** To add a layer of semantic meaning onto the golden data of an organization — centralizing and linking data into real-world objects (e.g., Employee, Asset, Order) so that applications, AI, and operators share a single consistent model.

---

**Q2. Which open data format does Foundry use by default for storing transformed data?**

> **A:** **Parquet** — a columnar, open-source format optimized for analytical queries.

---

**Q3. What is a Foundry "Branch" and why is it used?**

> **A:** A Branch is a copy-on-write snapshot of a dataset at a point in time. It enables safe, isolated experimentation — developers can build and test transforms on a branch without affecting the production (`master`) branch.

---

**Q4. What are "Datasets" in Foundry?**

> **A:** Datasets are the core storage unit in Foundry. They store versioned, immutable data — every write creates a new transaction, preserving full lineage history. Datasets can be raw (files) or tabular (schemas).

---

**Q5. What is the role of the Foundry Catalog?**

> **A:** The Catalog is the centralized metadata store for all Foundry resources (datasets, ontology objects, pipelines). It provides search, lineage visualization, and access control for every asset on the platform.

---

## Data Integration & Pipelines

**Q6. What is the correct sequence to configure a Direct Connection in Foundry?**

> **A:**
> 1. Configure network egress policy
> 2. Provision credentials
> 3. Create source in Data Connection
> 4. Configure network policy on the source system

---

**Q7. What is a Foundry Agent Host, and what is its minimum RAM requirement?**

> **A:** An Agent Host is a customer-managed compute node that runs inside the customer's network perimeter, enabling Foundry to pull data from private/on-prem systems. Minimum RAM: **16 GB**.

---

**Q8. Which two actions are essential for securing a Foundry Agent Host? (select two)**

> **A:**
> - Ensure the agent host can communicate with Palantir's cloud endpoints
> - Configure the firewall to block all traffic **except** to the desired Palantir destinations

---

**Q9. What role is required to configure network egress policies in Foundry?**

> **A:** **Information Security Officer (ISO)** — only users with this role can approve and configure egress rules.

---

**Q10. Which integration method is used to connect Azure Blob Storage directly to Foundry?**

> **A:** **Direct Connection** — no agent host is needed because Azure is a cloud-to-cloud connection using Foundry's native connectors.

---

**Q11. How does Foundry handle semi-structured data (e.g., JSON, XML) in Transforms?**

> **A:** By **leveraging custom Python or Java code** within Transforms to parse and flatten the semi-structured fields into tabular columns before further processing.

---

**Q12. What is a "Virtual Table" in Foundry AIP, and when is it used?**

> **A:** A Virtual Table exposes data from legacy or external systems as if it were a Foundry dataset — **without physically moving or copying the data**. It is used to integrate data from systems that cannot be easily replicated into Foundry.

---

## Ontology — Objects, Links & Actions

**Q13. What are "Object Types" in the Palantir Ontology?**

> **A:** Object Types define the schema for real-world entities (e.g., `Employee`, `Shipment`, `Equipment`). Each instance of an Object Type is an **Object** — equivalent to a row in a dataset but enriched with semantic meaning and relationships.

---

**Q14. What is a "Link Type" in the Ontology?**

> **A:** A Link Type defines a relationship between two Object Types (e.g., `Employee` *works on* `Project`). Links enable graph-style traversal — querying related objects across the ontology without writing joins.

---

**Q15. What are the two primary responsibilities of "Action Types" in the Palantir Ontology? (select two)**

> **A:**
> - **Capture data from operators** (e.g., form submissions, field updates)
> - **Orchestrate decision-making processes** (e.g., trigger downstream workflows or writes back to source systems)

---

**Q16. What does it mean for a pipeline to "back" an ontology object type?**

> **A:** The pipeline's output dataset is the source of truth for the object type — each row in the dataset maps to one object instance. Changes in the pipeline (new rows, updated fields) automatically propagate to the ontology.

---

**Q17. What is a "Property" on an Ontology object?**

> **A:** A Property is a typed attribute of an Object Type (e.g., `Employee.name: string`, `Order.amount: double`). Properties map directly to columns in the backing dataset.

---

## Workshop (Application Builder)

**Q18. What is Palantir Workshop?**

> **A:** Workshop is Foundry's low-code/no-code application builder. It lets operators and developers compose interactive operational applications — dashboards, forms, maps, tables — directly on top of ontology objects, without writing frontend code.

---

**Q19. What is the primary data source that Workshop applications use?**

> **A:** The **Ontology** — Workshop widgets bind to Object Types and their properties, actions, and links. This ensures all apps share the same consistent, governed view of data.

---

**Q20. What is a "Module" in Workshop?**

> **A:** A Module is a single page or screen within a Workshop application. Complex apps are composed of multiple modules, each focused on a specific workflow (e.g., an overview module, a detail module, an edit module).

---

**Q21. How do Workshop Actions differ from Ontology Action Types?**

> **A:** Ontology **Action Types** define *what* an action does (the logic, validation, write-back). Workshop **Action buttons** are the UI trigger that invokes a configured Action Type — Workshop is the interface layer, the Ontology is the logic layer.

---

## AIP (Artificial Intelligence Platform)

**Q22. What is Palantir AIP?**

> **A:** AIP is Palantir's layer for deploying large language models (LLMs) and AI agents **on top of the Ontology**. It allows organizations to run AI workflows with access to live, governed enterprise data — not just static documents.

---

**Q23. What is an "AIP Logic function"?**

> **A:** AIP Logic is a TypeScript-based function layer that runs close to the Ontology. It is used to define custom business logic, transformations, and AI-augmented workflows that can be invoked from Workshop, Actions, or external systems.

---

**Q24. Why is grounding AI models in the Palantir Ontology considered a best practice?**

> **A:** Because the Ontology provides **live, governed, semantic context** — the AI operates on up-to-date enterprise data with access controls enforced, reducing hallucinations and ensuring outputs are based on real organizational state.

---

**Q25. What is "AIP Assist" in Workshop?**

> **A:** AIP Assist is an AI copilot embedded in Workshop that helps operators complete tasks by suggesting actions, summarizing object data, or drafting responses — all grounded in the ontology objects visible in the current context.

---

## Fusion / Spreadsheet Integration

**Q26. A user synced a Fusion sheet to a Foundry dataset. Which three post-sync actions are available? (select three)**

> **A:**
> - Change the **branch** of the dataset
> - **Rename** the synced dataset
> - Modify the **export column type** to match the desired data types

---

**Q27. How can a user avoid synchronization conflicts when using a Fusion sheet backed by a Foundry dataset?**

> **A:** Use **table sync only** (not sheet sync) — mixing table sync and sheet sync in the same Fusion sheet causes conflicts because they write to different ranges simultaneously.

---

**Q28. What minimum permission level does a user need on a dataset to ensure their Fusion sheet changes reflect in that dataset?**

> **A:** **Editor** — Viewer permission allows reading but not writing back changes.

---

## Enterprise IT & Security

**Q29. How does Foundry fit into an Enterprise IT landscape?**

> **A:** Foundry acts as a **data integration and operating layer** sitting above existing source systems (ERP, CRM, IoT, databases). It ingests data via connectors, unifies it in the Ontology, and exposes it to applications and AI — without replacing source systems.

---

**Q30. What is the Foundry "Multipass" system?**

> **A:** Multipass is Palantir's identity and access management (IAM) layer. It handles authentication (SSO, MFA), authorization (role-based access control), and token management across all Foundry services — ensuring every data access is audited and permissioned.

---

## Key Data Flow to Remember

```
Source System
    → Data Connection / Agent Host
        → Raw Dataset (Parquet)
            → Transform Pipeline
                → Ontology (Object Types, Links, Actions)
                    → Workshop Apps / AIP Agents
```

---

## Quick Tips for the Exam

- The Foundry Aware exam is open to both technical and non-technical users — questions test **conceptual understanding**, not code syntax.
- Focus heavily on **Ontology concepts** (objects, links, actions, properties) — they underpin Workshop, AIP, and pipelines.
- Know the data flow: Source System → Data Connection / Agent Host → Dataset → Transform Pipeline → Ontology → Workshop/AIP.
- For security questions: think **least privilege**, **egress control**, and **Multipass**.
- Register and take the free exam at [learn.palantir.com/foundry-aip-aware](https://learn.palantir.com/foundry-aip-aware).

---

## Sources

- [Foundry & AIP Aware – Palantir Learn](https://learn.palantir.com/foundry-aip-aware)
- [Certification Exam Study Guides – Palantir Learn](https://learn.palantir.com/page/exam-guides)
- [Palantir Data Engineering Certification Q&A – Stuvia](https://www.stuvia.com/en-us/doc/7730961/palantir-data-engineering-certification-exam-questions-and-verified-answers-latest-update-20252026-graded-a.)
- [Palantir Data Engineering Flashcards – Quizlet](https://quizlet.com/991711798/palantir-data-engineering-certification-exam-flashcards/)
- [Palantir Foundry Ontology: Data Flow with AIP & Workshop – Udemy](https://www.udemy.com/course/palantir-foundry-ontology-data-flow-with-aip-workshop/)
