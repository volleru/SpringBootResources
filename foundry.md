# Palantir Foundry Aware Certification Exam - Q&A Prep

---

## Section 1: Foundry Core Concepts

**Q1. What is Palantir Foundry?**
> Palantir Foundry is an enterprise data platform that enables organizations to integrate, manage, analyze, and act on data. It connects raw data to operational decision-making through pipelines, ontologies, and applications.

---

**Q2. What are the three main layers of Foundry?**
> 1. **Data Layer** – Ingestion and storage of raw data (datasets, branches)
> 2. **Ontology Layer** – Semantic layer representing real-world objects, relationships, and actions
> 3. **Application Layer** – Tools like Workshop, Slate, and Quill that consume the ontology to build apps

---

**Q3. What is the Ontology in Foundry?**
> The Ontology is a semantic model that maps data to real-world business entities (Object Types), their attributes (Properties), relationships (Link Types), and operations (Actions). It acts as a shared language between data and applications.

---

**Q4. What is an Object Type?**
> An Object Type defines the schema of a real-world entity in the Ontology (e.g., `Employee`, `Order`, `Aircraft`). It has properties and can be linked to other object types.

---

**Q5. What is a Link Type?**
> A Link Type defines a relationship between two Object Types (e.g., `Employee` *belongs to* `Department`). Links can be one-to-one, one-to-many, or many-to-many.

---

**Q6. What is an Action in Foundry Ontology?**
> An Action is a defined operation that can create, update, or delete objects in the Ontology. Actions enforce business logic and write back to the underlying data source.

---

**Q7. What is an Object Set?**
> An Object Set is a filtered or grouped collection of objects of a given type. It is used in Workshop, Functions, and queries to work with subsets of ontology data.

---

**Q8. What is a Foundry Function?**
> A Function is custom TypeScript or Python logic registered in the Ontology that computes derived values, aggregations, or enables advanced querying on object sets.

---

**Q9. What is an Interface in the Ontology?**
> An Interface defines a shared structure (set of properties) that multiple Object Types can implement. This enables polymorphic querying across different but structurally similar object types.

---

## Section 2: Data Integration & Pipelines

**Q10. What is a Dataset in Foundry?**
> A Dataset is the fundamental unit of data storage in Foundry. It stores structured or unstructured data and is versioned using a branch/transaction model similar to git.

---

**Q11. What is Code Repository (Code Repo) in Foundry?**
> Code Repository is Foundry's integrated development environment for writing and managing data transformation code (Python, SQL, Java/Scala via Spark). It supports version control and CI/CD pipelines.

---

**Q12. What transformation tools are available in Foundry?**
> - **Contour** – No-code visual data exploration and transformation
> - **Code Repositories** – Python/SQL/Spark-based transforms
> - **Pipeline Builder** – Visual pipeline orchestration tool
> - **Fusion** – Low-code data preparation

---

**Q13. What is a Transform in Foundry?**
> A Transform is a unit of logic that takes one or more input datasets and produces one or more output datasets. Transforms are composable and versioned.

---

**Q14. What is a Branch in Foundry datasets?**
> Branches in Foundry are similar to git branches. They allow parallel development and experimentation on datasets without affecting the main/production data. Changes are merged via transactions.

---

**Q15. What is Magritte in Foundry?**
> Magritte is Foundry's data connection and ingestion framework. It provides connectors to external systems (databases, APIs, file systems) to bring data into Foundry.

---

## Section 3: Applications & Tools

**Q16. What is Workshop in Foundry?**
> Workshop is Foundry's no-code/low-code application builder. It allows users to build interactive operational applications on top of the Ontology without writing frontend code.

---

**Q17. What is Slate in Foundry?**
> Slate is a flexible web application framework in Foundry that allows developers to build custom dashboards and applications using HTML, CSS, and JavaScript with access to Foundry data.

---

**Q18. What is Contour used for?**
> Contour is a no-code data exploration and analysis tool. Users can visually filter, aggregate, join, and visualize datasets without writing code.

---

**Q19. What is Quill in Foundry?**
> Quill is Foundry's document and report builder that allows users to create data-driven narratives and reports combining text with live Foundry data visualizations.

---

**Q20. What is AIP (Artificial Intelligence Platform) in Foundry?**
> AIP is Palantir's AI layer built on top of Foundry that enables organizations to deploy and operationalize LLMs and AI models securely within their enterprise data environment.

---

## Section 4: Security & Governance

**Q21. What is Marking in Foundry?**
> Markings are access control labels applied to datasets or objects to restrict visibility to authorized users or groups. They enforce data governance and compliance requirements.

---

**Q22. What are Categories in Foundry security?**
> Categories are organizational groupings used to classify data sensitivity and enforce access policies. They work alongside Markings to provide fine-grained data access control.

---

**Q23. What is the role of the Foundry Compass?**
> Compass is Foundry's project and resource management tool. It organizes datasets, transforms, ontology objects, and applications into projects with role-based access control.

---

**Q24. What is a Gotham vs Foundry difference?**
> - **Gotham** – Palantir's intelligence/defense platform focused on structured data analysis for government/defense use cases
> - **Foundry** – Palantir's enterprise commercial platform focused on operational data integration and decision-making for businesses

---

## Section 5: Ontology Backed Applications

**Q25. What makes an application "Ontology-backed"?**
> An Ontology-backed application reads from and writes to the Ontology layer instead of directly querying raw datasets. This ensures consistency, reusability, and enforced business logic across all applications.

---

**Q26. What is the difference between a Property and a Derived Property?**
> - **Property** – Directly mapped from a column in the backing dataset
> - **Derived Property** – Computed via a Function using logic applied to existing properties (e.g., calculating age from birthdate)

---

**Q27. What is a Sync in Foundry Ontology?**
> A Sync (Object Type Sync) is the configuration that maps dataset columns to Object Type properties, keeping the Ontology in sync with the underlying dataset as data changes.

---

**Q28. What is the purpose of the Ontology Manager?**
> Ontology Manager is the tool in Foundry used to create, configure, and manage Object Types, Link Types, Actions, Functions, and Interfaces. It is the central UI for ontology development.

---

## Section 6: Scenario-Based Questions

**Q29. A business analyst wants to explore a dataset without writing code. Which Foundry tool should they use?**
> **Contour** – It provides a no-code, visual interface for data exploration, filtering, and aggregation.

---

**Q30. A developer needs to write a Python-based data transformation that runs on a schedule. What should they use?**
> **Code Repository** with a Python transform, scheduled via Foundry's **Build Scheduler** or **pipeline triggers**.

---

**Q31. You need to allow an external database to be ingested into Foundry. What component handles this?**
> **Magritte** – Foundry's connector framework for ingesting data from external sources.

---

**Q32. A team wants to build an operational app where users can update order statuses. What Foundry components are needed?**
> - **Object Type** for `Order`
> - **Action** to update order status
> - **Workshop** application to expose the UI to users

---

**Q33. How does Foundry ensure data lineage?**
> Foundry automatically tracks **data lineage** through its pipeline graph. Every dataset knows its upstream inputs and downstream consumers, making it easy to trace data origin and impact of changes.

---

## Quick Reference Cheat Sheet

| Term | Definition |
|---|---|
| Object Type | Schema for a real-world entity |
| Property | Attribute of an Object Type |
| Link Type | Relationship between Object Types |
| Action | Write operation on Ontology objects |
| Function | Custom compute logic (TypeScript/Python) |
| Interface | Shared structure across Object Types |
| Object Set | Filtered collection of objects |
| Dataset | Core data storage unit in Foundry |
| Transform | Logic unit producing output datasets |
| Workshop | No-code app builder |
| Contour | No-code data exploration tool |
| Slate | Custom web app framework |
| Magritte | Data ingestion/connector framework |
| Marking | Data access control label |
| Compass | Project & resource manager |

---

*Good luck with your Foundry Aware Certification!*
