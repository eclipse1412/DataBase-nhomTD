1. Introduction & Project ScopeSystem Objective: Establish a centralized database for a university research division to seamlessly manage four key domains: Research Grants, Equipment, Chemical Inventories, and Personnel.
2. Core Problem Solved: Links grant funding directly to asset purchases, guarantees laboratory safety by blocking uncertified access at the database level, and maintains an immutable audit trail for compliance.Business
3. Rules:Equipment must belong to exactly one lab and (if funded) one research grant.Only faculty members can serve as Principal Investigators (PIs).Hazardous chemicals strictly require valid, non-expired safety certifications for access.
4. Certifications automatically lapse upon expiration.
5. The access log (AccessLog) is strictly append-only; UPDATE and DELETE operations are completely forbidden
6. Database DesignConceptual Model (ER Diagram): Defines 10 core entities (Department, Personnel, Lab, ResearchGrant, Certification, PersonnelCertification, Equipment, ChemicalInventory, EquipmentReservation, AccessLog) along with their 1:N and M:N relationships.
7. Logical Schema Mapping: Translates conceptual entities into 10 relational tables, establishing Primary Keys (PKs) for unique identification and Foreign Keys (FKs) for table relationships.
8. Normalization Verification (1NF $\rightarrow$ BCNF):1NF: Eliminates multi-valued attributes (e.g., repeating certification lists) by splitting them into atomic, single-valued rows.
9. 2NF: Removes partial functional dependencies from tables with composite keys.
10. 3NF: Removes transitive dependencies by decoupling department, lab, and grant details into independent entities.
11. BCNF: Ensures every determinant ($X \rightarrow Y$) is a candidate key, eliminating insertion, update, and deletion anomalies.
12. Data DictionaryTechnical Specifications: Comprehensive breakdown of all 10 tables and their respective attribute columns.
13. Data Types: Defines precise storage formats for each field (e.g., INT, VARCHAR, DECIMAL for currency, DATE/TIMESTAMP, and ENUM for fixed status values).Data Constraints: Enforces integrity using NOT NULL, UNIQUE, Foreign Key relationships, and CHECK rules (e.g., budget > 0, biosafety_level BETWEEN 1 AND 4).
14. ConclusionValue Delivered: Achieves financial transparency, optimizes asset management, and enforces strict lab safety compliance directly at the database level.Future Work: Lays a fully normalized database foundation ready for backend API integration and frontend user interface development in subsequent phases.
