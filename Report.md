Database Project Report
Due Date: [29/08 – Week 3]
Project ID & Title: #4: University Research & Lab Asset Tracker

________________________________________
A. Project Identity
Field	Value
Team Name	TD
Team Members	Trần Anh Hào (n24dece068@student.ptithcm.edu.vn)
Chu Tự Đức(n24dece063@student.ptithcm.edu.vn)
Nguyễn Anh Tuân (n24dece101@student.ptithcm.edu.vn)

Project Title	SafeLab: An Integrated System for Research Grant, Equipment, Chemical Inventory, and Access Control Management

________________________________________
B. Report Structure
1. Introduction & Project Scope
(Adapted from ISO/IEC/IEEE 29148)
1.1 System Objective
The University Research & Lab Asset Tracker (hereafter SafeLab) is designed to give a university research division a single system of record for everything that moves through its laboratories: the grants that fund the work, the equipment purchased under those grants, the chemical inventory stored on-site, and the people — students, faculty, and staff — who interact with all of the above.

The system's primary objectives are:


1.	Financial traceability — link every piece of equipment and every chemical purchase back to the research grant that funded it, so that grant expenditure can be audited against the funding agency's terms.
2.	Asset visibility — provide real-time status (available, in use, under maintenance, retired) for every tracked piece of equipment across all labs in a department.
3.	Safety compliance — guarantee, at the database level, that hazardous chemicals and controlled equipment can only be accessed, checked out, or logged as "entered" by personnel holding a current, unexpired safety certification appropriate to that hazard class. This is the core differentiator of the system and is enforced with SQL triggers rather than relying on application-layer checks alone.
4.	Accountability — maintain an immutable audit trail (access log) of who accessed which lab, equipment, or chemical, and when, including any access attempts that were denied for lack of certification.
5.	Lifecycle management — track certification issue/expiry dates, chemical expiry dates, and equipment maintenance/retirement dates so the system can proactively flag anything that is about to lapse.
1.2 Business Rules & Constraints
#	Rule
BR-1	Every piece of equipment must be associated with exactly one lab and, if purchased with grant funding, with exactly one research grant.
BR-2	A research grant must have a designated Principal Investigator (PI), who must be a person with role faculty.
BR-3	A chemical inventory item with a hazard class other than none must specify a minimum required certification.
BR-4	Hazardous materials may only be accessed (checked out, viewed, or physically entered into storage) by personnel holding a valid, non-expired certification matching the chemical's required hazard class. Any access attempt by uncertified personnel must be rejected at the database level and still recorded in the access log as authorized = FALSE.
BR-5	A certification expires automatically after its defined validity period; the system must never treat an expired certification as valid, even if the row still exists in the certification-history table.
BR-6	Equipment reservations must not overlap for the same piece of equipment (no double-booking).
BR-7	A grant's cumulative equipment/chemical purchase value must not exceed its approved budget.
BR-8	A chemical inventory item past its expiry date may not be checked out for use (only disposal-related access is permitted).
BR-9	Only users with the lab_manager or admin database role may insert, update, or delete rows in Certification, PersonnelCertification, or ChemicalInventory.
BR-10	The access log is append-only: no UPDATE or DELETE is permitted on AccessLog by any application role, to preserve audit integrity.

________________________________________
2. Database Design
(ISO/IEC 19505 / IE Standards)
2.1 Conceptual Model (ER/EER Diagram)
Entities


•	Department
•	Personnel (faculty, postdoc, grad_student, undergrad_student, staff)
•	Lab
•	ResearchGrant
•	Certification
•	PersonnelCertification (associative entity: many-to-many between Personnel and Certification, time-stamped)
•	Equipment
•	ChemicalInventory
•	EquipmentReservation
•	AccessLog

Relationships


•	Department 1 — N Lab
•	Department 1 — N Personnel
•	ResearchGrant N — 1 Personnel (PI, role = faculty)
•	Lab 1 — N Equipment
•	Lab 1 — N ChemicalInventory
•	ResearchGrant 1 — N Equipment (equipment purchased under a grant, optional)
•	ResearchGrant 1 — N ChemicalInventory (optional)
•	Personnel M — N Certification (via PersonnelCertification)
•	Certification 1 — N ChemicalInventory (min_required_cert_id — the certification needed to legally access that chemical)
•	Personnel 1 — N EquipmentReservation, Equipment 1 — N EquipmentReservation
•	Personnel 1 — N AccessLog, Lab 1 — N AccessLog

Mermaid ER Diagram (for rendering with any Mermaid-compatible viewer):

erDiagram

    DEPARTMENT ||--o{ LAB : contains

    DEPARTMENT ||--o{ PERSONNEL : employs

    PERSONNEL ||--o{ RESEARCH_GRANT : "is PI of"

    LAB ||--o{ EQUIPMENT : houses

    LAB ||--o{ CHEMICAL_INVENTORY : stores

    RESEARCH_GRANT ||--o{ EQUIPMENT : funds

    RESEARCH_GRANT ||--o{ CHEMICAL_INVENTORY : funds

    CERTIFICATION ||--o{ CHEMICAL_INVENTORY : "required for"

    PERSONNEL ||--o{ PERSONNEL_CERTIFICATION : holds

    CERTIFICATION ||--o{ PERSONNEL_CERTIFICATION : "granted as"

    PERSONNEL ||--o{ EQUIPMENT_RESERVATION : books

    EQUIPMENT ||--o{ EQUIPMENT_RESERVATION : "is booked in"

    PERSONNEL ||--o{ ACCESS_LOG : generates

    LAB ||--o{ ACCESS_LOG : records

Cardinality notes: the ResearchGrant → Equipment/ChemicalInventory link is optional (0..N) because a lab may also hold department-owned assets not tied to any grant. The Certification → ChemicalInventory link is mandatory only when hazard_class <> 'none'.
2.2 Logical Schema Mapping
Entity/Relationship	Relational Table	Primary Key	Foreign Keys
Department	department	dept_id	—
Personnel	personnel	person_id	dept_id → department
Lab	lab	lab_id	dept_id → department
ResearchGrant	research_grant	grant_id	pi_person_id → personnel
Certification	certification	cert_id	—
Personnel–Certification (M:N)	personnel_certification	(person_id, cert_id, issue_date)	person_id → personnel, cert_id → certification
Equipment	equipment	equipment_id	lab_id → lab, grant_id → research_grant
ChemicalInventory	chemical_inventory	chemical_id	lab_id → lab, grant_id → research_grant, min_required_cert_id → certification
EquipmentReservation	equipment_reservation	reservation_id	equipment_id → equipment, person_id → personnel
AccessLog	access_log	log_id	person_id → personnel, lab_id → lab
2.3 Normalization Verification (1NF → 3NF/BCNF)
Starting point — an unnormalized "flat" equipment record as it might arrive from a spreadsheet:

Equipment(equipment_id, equipment_name, lab_id, lab_name, dept_id, dept_name,

          grant_id, grant_title, pi_name, cert_names_required)

Step 1 — 1NF. cert_names_required is a repeating/multi-valued group (a chemical or piece of equipment can require more than one certification). We remove the repeating group into its own row-per-value table, giving every attribute an atomic value:

Equipment(equipment_id, equipment_name, lab_id, lab_name, dept_id, dept_name, grant_id, grant_title, pi_name)

EquipmentCertRequirement(equipment_id, cert_id)

→ Now in 1NF.

Step 2 — 2NF. The table has a single-column primary key (equipment_id), so there is no partial dependency on part of a composite key. EquipmentCertRequirement(equipment_id, cert_id) is already fully dependent on its whole composite key. → Both tables satisfy 2NF.

Step 3 — 3NF. In Equipment, examine functional dependencies:


•	equipment_id → lab_id, lab_name, dept_id, dept_name, grant_id, grant_title, pi_name
•	but also: lab_id → lab_name, dept_id, dept_name (transitive: lab attributes depend on lab_id, not directly on equipment_id)
•	and: grant_id → grant_title, pi_name (transitive: grant attributes depend on grant_id, not directly on equipment_id)

These are transitive dependencies (non-key attribute → non-key attribute), which violate 3NF. We decompose:

Equipment(equipment_id, equipment_name, lab_id, grant_id)

Lab(lab_id, lab_name, dept_id)

Department(dept_id, dept_name)

ResearchGrant(grant_id, grant_title, pi_person_id)

Personnel(person_id, pi_name, ...)

Now every non-key attribute depends only on its own table's primary key. → 3NF achieved.

Step 4 — BCNF check. For every functional dependency X → Y in the decomposed tables, X must be a superkey.


•	Equipment: equipment_id → equipment_name, lab_id, grant_id. equipment_id is the sole candidate key.  
•	Lab: lab_id → lab_name, dept_id. lab_id is the sole candidate key. 
•	PersonnelCertification(person_id, cert_id, issue_date) → expiry_date: the determinant (person_id, cert_id, issue_date) is the composite candidate key itself
•	ChemicalInventory: chemical_id → name, cas_number, hazard_class, lab_id, min_required_cert_id, expiry_date. chemical_id is the sole key, and cas_number is a candidate key too (unique per chemical), but cas_number → name, hazard_class does not violate BCNF because cas_number is itself a candidate key (superkey). 

All relations are therefore in BCNF with no remaining anomalies.

________________________________________
3. Data Dictionary
(Adapted from ISO/IEC 11179)

Table: department

Attribute	Type	Constraints	Description
dept_id	INT	PK, AUTO_INCREMENT	Unique department identifier
dept_name	VARCHAR(100)	NOT NULL, UNIQUE	Department name
building	VARCHAR(50)	NOT NULL	Building housing the department

Table: personnel

Attribute	Type	Constraints	Description
person_id	INT	PK, AUTO_INCREMENT	Unique person identifier
first_name	VARCHAR(50)	NOT NULL	First name
last_name	VARCHAR(50)	NOT NULL	Last name
Email	VARCHAR(120)	NOT NULL, UNIQUE	University email
Role	ENUM('faculty','postdoc','grad_student','undergrad_student','staff')	NOT NULL	Role determines default access privileges
dept_id	INT	FK → department	Home department

Table: research_grant

Attribute	Type	Constraints	Description
grant_id	INT	PK, AUTO_INCREMENT	Unique grant identifier
Title	VARCHAR(200)	NOT NULL	Grant title
funding_agency	VARCHAR(150)	NOT NULL	Sponsoring agency
budget	DECIMAL(12,2)	NOT NULL, CHECK (budget > 0)	Approved budget (USD)
start_date	DATE	NOT NULL	Grant start date
end_date	DATE	NOT NULL, CHECK (end_date > start_date)	Grant end date
pi_person_id	INT	FK → personnel, NOT NULL	Principal Investigator

Table: lab

Attribute	Type	Constraints	Description
lab_id	INT	PK, AUTO_INCREMENT	Unique lab identifier
lab_name	VARCHAR(100)	NOT NULL	Lab name
dept_id	INT	FK → department, NOT NULL	Owning department
room_number	VARCHAR(20)	NOT NULL	Physical room
biosafety_level	SMALLINT	CHECK (biosafety_level BETWEEN 1 AND 4)	BSL rating of the lab

Table: certification

Attribute	Type	Constraints	Description
cert_id	INT	PK, AUTO_INCREMENT	Unique certification identifier
cert_name	VARCHAR(100)	NOT NULL	e.g. "Radioactive Materials Handling"
hazard_class	VARCHAR(50)	NOT NULL	Hazard category this cert covers
valid_period_months	SMALLINT	NOT NULL, CHECK (valid_period_months > 0)	Validity length

Table: personnel_certification

Attribute	Type	Constraints	Description
person_id	INT	PK(part), FK → personnel	Certified person
cert_id	INT	PK(part), FK → certification	Certification held
issue_date	DATE	PK(part), NOT NULL	Date issued
expiry_date	DATE	NOT NULL	Computed/stored expiry (issue_date + valid_period_months)

Table: equipment

Attribute	Type	Constraints	Description
equipment_id	INT	PK, AUTO_INCREMENT	Unique equipment identifier
equipment_name	VARCHAR(150)	NOT NULL	Name/model
category	VARCHAR(80)	NOT NULL	e.g. "Centrifuge", "Spectrometer"
lab_id	INT	FK → lab, NOT NULL	Home lab
grant_id	INT	FK → research_grant, NULLABLE	Funding grant, if any
purchase_date	DATE	NOT NULL	Purchase date
Value	DECIMAL(10,2)	NOT NULL, CHECK (value >= 0)	Purchase value
Status	ENUM('available','in_use','maintenance','retired')	NOT NULL DEFAULT 'available'	Current status

Table: chemical_inventory

Attribute	Type	Constraints	Description
Chemi cal_id	INT	PK, AUTO_INCREMENT	Unique chemical record identifier
name	VARCHAR(150)	NOT NULL	Chemical name
cas_number	VARCHAR(20)	UNIQUE	CAS registry number
hazard_class	VARCHAR(50)	NOT NULL DEFAULT 'none'	e.g. "flammable", "carcinogen", "radioactive", "none"
quantity	DECIMAL(10,3)	NOT NULL, CHECK (quantity >= 0)	Amount on hand
unit	VARCHAR(20)	NOT NULL	e.g. "L", "kg"
lab_id	INT	FK → lab, NOT NULL	Storage lab
grant_id	INT	FK → research_grant, NULLABLE	Funding grant, if any
storage_location	VARCHAR(100)	NOT NULL	Cabinet/shelf code
expiry_date	DATE	NOT NULL	Expiry date
min_required_cert_id	INT	FK → certification, NULLABLE	Certification required to access (NULL if hazard_class = 'none')

Table: equipment_reservation

Attribute	Type	Constraints	Description
reservation_id	INT	PK, AUTO_INCREMENT	Unique reservation identifier
equipment_id	INT	FK → equipment, NOT NULL	Reserved equipment
person_id	INT	FK → personnel, NOT NULL	Reserving person
start_time	TIMESTAMP	NOT NULL	Reservation start
end_time	TIMESTAMP	NOT NULL, CHECK (end_time > start_time)	Reservation end
purpose	VARCHAR(200)		Stated purpose

Table: access_log

Attribute	Type	Constraints	Description
log_id	BIGINT	PK, AUTO_INCREMENT	Unique log entry
person_id	INT	FK → personnel, NOT NULL	Person attempting access
lab_id	INT	FK → lab, NOT NULL	Lab where access occurred
resource_type	ENUM('equipment','chemical','lab_entry')	NOT NULL	Type of resource accessed
resource_id	INT	NULLABLE	equipment_id or chemical_id, if applicable
access_time	TIMESTAMP	NOT NULL DEFAULT CURRENT_TIMESTAMP	When access occurred/was attempted
access_type	ENUM('checkout','checkin','view','entry')	NOT NULL	Nature of the access
authorized	BOOLEAN	NOT NULL	Whether the system permitted the access

________________________________________
4. Database Implementation
(SQL Style Guide Compliant — PostgreSQL dialect)
4.1 DDL Script (Tables, Views, Indexes, Triggers)
-- ============================================================

-- CORE TABLES

-- ============================================================

CREATE TABLE department (

    dept_id     SERIAL PRIMARY KEY,

    dept_name   VARCHAR(100) NOT NULL UNIQUE,

    building    VARCHAR(50)  NOT NULL

);

CREATE TABLE personnel (

    person_id   SERIAL PRIMARY KEY,

    first_name  VARCHAR(50)  NOT NULL,

    last_name   VARCHAR(50)  NOT NULL,

    email       VARCHAR(120) NOT NULL UNIQUE,

    role        VARCHAR(20)  NOT NULL

                CHECK (role IN ('faculty','postdoc','grad_student','undergrad_student','staff')),

    dept_id     INT REFERENCES department(dept_id)

);

CREATE TABLE research_grant (

    grant_id        SERIAL PRIMARY KEY,

    title           VARCHAR(200) NOT NULL,

    funding_agency  VARCHAR(150) NOT NULL,

    budget          DECIMAL(12,2) NOT NULL CHECK (budget > 0),

    start_date      DATE NOT NULL,

    end_date        DATE NOT NULL,

    pi_person_id    INT NOT NULL REFERENCES personnel(person_id),

    CHECK (end_date > start_date)

);

CREATE TABLE lab (

    lab_id           SERIAL PRIMARY KEY,

    lab_name         VARCHAR(100) NOT NULL,

    dept_id          INT NOT NULL REFERENCES department(dept_id),

    room_number      VARCHAR(20) NOT NULL,

    biosafety_level  SMALLINT CHECK (biosafety_level BETWEEN 1 AND 4)

);

CREATE TABLE certification (

    cert_id             SERIAL PRIMARY KEY,

    cert_name           VARCHAR(100) NOT NULL,

    hazard_class        VARCHAR(50)  NOT NULL,

    valid_period_months SMALLINT NOT NULL CHECK (valid_period_months > 0)

);

CREATE TABLE personnel_certification (

    person_id    INT NOT NULL REFERENCES personnel(person_id),

    cert_id      INT NOT NULL REFERENCES certification(cert_id),

    issue_date   DATE NOT NULL,

    expiry_date  DATE NOT NULL,

    PRIMARY KEY (person_id, cert_id, issue_date),

    CHECK (expiry_date > issue_date)

);

CREATE TABLE equipment (

    equipment_id    SERIAL PRIMARY KEY,

    equipment_name  VARCHAR(150) NOT NULL,

    category        VARCHAR(80)  NOT NULL,

    lab_id          INT NOT NULL REFERENCES lab(lab_id),

    grant_id        INT REFERENCES research_grant(grant_id),

    purchase_date   DATE NOT NULL,

    value           DECIMAL(10,2) NOT NULL CHECK (value >= 0),

    status          VARCHAR(20) NOT NULL DEFAULT 'available'

                    CHECK (status IN ('available','in_use','maintenance','retired'))

);

CREATE TABLE chemical_inventory (

    chemical_id           SERIAL PRIMARY KEY,

    name                  VARCHAR(150) NOT NULL,

    cas_number            VARCHAR(20) UNIQUE,

    hazard_class          VARCHAR(50) NOT NULL DEFAULT 'none',

    quantity              DECIMAL(10,3) NOT NULL CHECK (quantity >= 0),

    unit                  VARCHAR(20) NOT NULL,

    lab_id                INT NOT NULL REFERENCES lab(lab_id),

    grant_id              INT REFERENCES research_grant(grant_id),

    storage_location      VARCHAR(100) NOT NULL,

    expiry_date           DATE NOT NULL,

    min_required_cert_id  INT REFERENCES certification(cert_id),

    CHECK (

        (hazard_class = 'none' AND min_required_cert_id IS NULL)

        OR (hazard_class <> 'none' AND min_required_cert_id IS NOT NULL)

    )

);

CREATE TABLE equipment_reservation (

    reservation_id  SERIAL PRIMARY KEY,

    equipment_id    INT NOT NULL REFERENCES equipment(equipment_id),

    person_id       INT NOT NULL REFERENCES personnel(person_id),

    start_time      TIMESTAMP NOT NULL,

    end_time        TIMESTAMP NOT NULL,

    purpose         VARCHAR(200),

    CHECK (end_time > start_time)

);

CREATE TABLE access_log (

    log_id         BIGSERIAL PRIMARY KEY,

    person_id      INT NOT NULL REFERENCES personnel(person_id),

    lab_id         INT NOT NULL REFERENCES lab(lab_id),

    resource_type  VARCHAR(20) NOT NULL CHECK (resource_type IN ('equipment','chemical','lab_entry')),

    resource_id    INT,

    access_time    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    access_type    VARCHAR(20) NOT NULL CHECK (access_type IN ('checkout','checkin','view','entry')),

    authorized     BOOLEAN NOT NULL

);

-- ============================================================

-- INDEXES

-- ============================================================

CREATE INDEX idx_equipment_lab        ON equipment(lab_id);

CREATE INDEX idx_equipment_grant      ON equipment(grant_id);

CREATE INDEX idx_chemical_lab         ON chemical_inventory(lab_id);

CREATE INDEX idx_chemical_expiry      ON chemical_inventory(expiry_date);

CREATE INDEX idx_personcert_expiry    ON personnel_certification(person_id, cert_id, expiry_date);

CREATE INDEX idx_accesslog_person     ON access_log(person_id, access_time);

CREATE INDEX idx_reservation_equip    ON equipment_reservation(equipment_id, start_time, end_time);

-- ============================================================

-- VIEWS

-- ============================================================

-- Currently valid (non-expired) certifications per person

CREATE VIEW v_active_certifications AS

SELECT pc.person_id, pc.cert_id, c.cert_name, c.hazard_class, pc.expiry_date

FROM personnel_certification pc

JOIN certification c ON c.cert_id = pc.cert_id

WHERE pc.expiry_date >= CURRENT_DATE;

-- Chemicals nearing expiry (within 30 days) for lab manager dashboards

CREATE VIEW v_chemicals_expiring_soon AS

SELECT chemical_id, name, lab_id, expiry_date

FROM chemical_inventory

WHERE expiry_date BETWEEN CURRENT_DATE AND CURRENT_DATE + INTERVAL '30 days';

-- Grant spend summary

CREATE VIEW v_grant_spend_summary AS

SELECT g.grant_id, g.title, g.budget,

       COALESCE(SUM(e.value), 0) AS equipment_spend,

       g.budget - COALESCE(SUM(e.value), 0) AS remaining_budget

FROM research_grant g

LEFT JOIN equipment e ON e.grant_id = g.grant_id

GROUP BY g.grant_id, g.title, g.budget;

-- ============================================================

-- TRIGGERS

-- ============================================================

-- (1) Auto-compute certification expiry_date on insert

CREATE OR REPLACE FUNCTION fn_set_cert_expiry()

RETURNS TRIGGER AS $$

DECLARE

    v_months SMALLINT;

BEGIN

    SELECT valid_period_months INTO v_months

    FROM certification WHERE cert_id = NEW.cert_id;

    NEW.expiry_date := NEW.issue_date + (v_months || ' months')::INTERVAL;

    RETURN NEW;

END;

$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_set_cert_expiry

BEFORE INSERT ON personnel_certification

FOR EACH ROW

EXECUTE FUNCTION fn_set_cert_expiry();

-- (2) CORE SAFETY TRIGGER — enforce that hazardous chemicals are only

--     accessed by personnel with a matching, unexpired certification.

--     Applied on INSERT into access_log for resource_type = 'chemical'.

CREATE OR REPLACE FUNCTION fn_enforce_hazmat_access()

RETURNS TRIGGER AS $$

DECLARE

    v_hazard_class  VARCHAR(50);

    v_required_cert INT;

    v_has_valid_cert BOOLEAN;

BEGIN

    IF NEW.resource_type = 'chemical' AND NEW.access_type IN ('checkout','entry') THEN

        SELECT hazard_class, min_required_cert_id

        INTO v_hazard_class, v_required_cert

        FROM chemical_inventory

        WHERE chemical_id = NEW.resource_id;

        IF v_hazard_class IS NOT NULL AND v_hazard_class <> 'none' THEN

            SELECT EXISTS (

                SELECT 1

                FROM personnel_certification

                WHERE person_id = NEW.person_id

                  AND cert_id   = v_required_cert

                  AND expiry_date >= CURRENT_DATE

            ) INTO v_has_valid_cert;

            IF NOT v_has_valid_cert THEN

                -- Record the attempt as unauthorized rather than silently

                -- blocking it, so the denial itself becomes part of the audit trail.

                NEW.authorized := FALSE;

                RAISE EXCEPTION

                    'Access denied: person_id % lacks a valid certification (cert_id %) for hazard class % on chemical_id %',

                    NEW.person_id, v_required_cert, v_hazard_class, NEW.resource_id;

            ELSE

                NEW.authorized := TRUE;

            END IF;

        ELSE

            NEW.authorized := TRUE; -- non-hazardous, no cert required

        END IF;

    END IF;

    RETURN NEW;

END;

$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_enforce_hazmat_access

BEFORE INSERT ON access_log

FOR EACH ROW

EXECUTE FUNCTION fn_enforce_hazmat_access();

-- (3) Prevent checkout of expired chemicals (BR-8)

CREATE OR REPLACE FUNCTION fn_block_expired_chemical_checkout()

RETURNS TRIGGER AS $$

DECLARE

    v_expiry DATE;

BEGIN

    IF NEW.resource_type = 'chemical' AND NEW.access_type = 'checkout' THEN

        SELECT expiry_date INTO v_expiry

        FROM chemical_inventory WHERE chemical_id = NEW.resource_id;

        IF v_expiry < CURRENT_DATE THEN

            RAISE EXCEPTION 'Access denied: chemical_id % expired on %', NEW.resource_id, v_expiry;

        END IF;

    END IF;

    RETURN NEW;

END;

$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_block_expired_chemical

BEFORE INSERT ON access_log

FOR EACH ROW

EXECUTE FUNCTION fn_block_expired_chemical_checkout();

-- (4) Prevent overlapping equipment reservations (BR-6)

CREATE OR REPLACE FUNCTION fn_prevent_reservation_overlap()

RETURNS TRIGGER AS $$

BEGIN

    IF EXISTS (

        SELECT 1 FROM equipment_reservation

        WHERE equipment_id = NEW.equipment_id

          AND reservation_id <> COALESCE(NEW.reservation_id, -1)

          AND (NEW.start_time, NEW.end_time) OVERLAPS (start_time, end_time)

    ) THEN

        RAISE EXCEPTION 'Reservation overlap for equipment_id %', NEW.equipment_id;

    END IF;

    RETURN NEW;

END;

$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_prevent_reservation_overlap

BEFORE INSERT OR UPDATE ON equipment_reservation

FOR EACH ROW

EXECUTE FUNCTION fn_prevent_reservation_overlap();

-- (5) Block UPDATE/DELETE on access_log to preserve audit integrity (BR-10)

CREATE OR REPLACE FUNCTION fn_block_access_log_mutation()

RETURNS TRIGGER AS $$

BEGIN

    RAISE EXCEPTION 'access_log is append-only; % is not permitted', TG_OP;

END;

$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_block_access_log_update

BEFORE UPDATE OR DELETE ON access_log

FOR EACH ROW

EXECUTE FUNCTION fn_block_access_log_mutation();
4.2 Advanced Queries & Performance Test Cases
-- Q1: All hazardous chemicals a given person is currently NOT certified to access

--     (useful for a lab manager verifying onboarding status).

SELECT ci.chemical_id, ci.name, ci.hazard_class

FROM chemical_inventory ci

WHERE ci.hazard_class <> 'none'

  AND NOT EXISTS (

      SELECT 1 FROM personnel_certification pc

      WHERE pc.person_id = :target_person_id

        AND pc.cert_id = ci.min_required_cert_id

        AND pc.expiry_date >= CURRENT_DATE

  );

-- Q2: Equipment utilization — total reserved hours per item over the last 90 days,

--     using a window function to also rank equipment within its lab.

SELECT

    e.equipment_id,

    e.equipment_name,

    e.lab_id,

    SUM(EXTRACT(EPOCH FROM (er.end_time - er.start_time)) / 3600) AS total_hours,

    RANK() OVER (

        PARTITION BY e.lab_id

        ORDER BY SUM(EXTRACT(EPOCH FROM (er.end_time - er.start_time)) / 3600) DESC

    ) AS usage_rank_in_lab

FROM equipment e

JOIN equipment_reservation er ON er.equipment_id = e.equipment_id

WHERE er.start_time >= CURRENT_DATE - INTERVAL '90 days'

GROUP BY e.equipment_id, e.equipment_name, e.lab_id;

-- Q3: Grant compliance report — grants where equipment + chemical spend

--     exceeds 90% of budget (early warning).

SELECT g.grant_id, g.title, g.budget,

       COALESCE(eq.eq_spend, 0) + COALESCE(ch.ch_spend, 0) AS total_spend,

       ROUND(100.0 * (COALESCE(eq.eq_spend,0) + COALESCE(ch.ch_spend,0)) / g.budget, 1) AS pct_used

FROM research_grant g

LEFT JOIN (

    SELECT grant_id, SUM(value) AS eq_spend FROM equipment GROUP BY grant_id

) eq ON eq.grant_id = g.grant_id

LEFT JOIN (

    -- Placeholder: assumes a chemical_cost column would be added in a cost-tracking extension

    SELECT grant_id, 0 AS ch_spend FROM chemical_inventory GROUP BY grant_id

) ch ON ch.grant_id = g.grant_id

WHERE (COALESCE(eq.eq_spend,0) + COALESCE(ch.ch_spend,0)) / g.budget > 0.9;

-- Q4: Unauthorized access attempts in the last 7 days (safety officer dashboard)

SELECT al.log_id, p.first_name, p.last_name, al.resource_type, al.resource_id, al.access_time

FROM access_log al

JOIN personnel p ON p.person_id = al.person_id

WHERE al.authorized = FALSE

  AND al.access_time >= CURRENT_TIMESTAMP - INTERVAL '7 days'

ORDER BY al.access_time DESC;

-- Performance test: verify the planner uses idx_personcert_expiry rather

-- than a sequential scan for the certification-check subquery in the trigger.

EXPLAIN ANALYZE

SELECT 1 FROM personnel_certification

WHERE person_id = 42 AND cert_id = 3 AND expiry_date >= CURRENT_DATE;

________________________________________
5. Verification & Security
5.1 Test Cases Showing Constraints Prevent Bad Data Insertion
-- TEST 1: Attempt to log a hazardous-chemical checkout by an uncertified person.

-- Expected result: transaction fails with the hazmat trigger's RAISE EXCEPTION.

INSERT INTO access_log (person_id, lab_id, resource_type, resource_id, access_type, authorized)

VALUES (17, 3, 'chemical', 8, 'checkout', TRUE);

-- >>> ERROR: Access denied: person_id 17 lacks a valid certification (cert_id 5)

--     for hazard class radioactive on chemical_id 8

-- TEST 2: Attempt to check out a chemical past its expiry date.

-- Expected result: rejected by trg_block_expired_chemical.

INSERT INTO access_log (person_id, lab_id, resource_type, resource_id, access_type, authorized)

VALUES (4, 2, 'chemical', 11, 'checkout', TRUE);

-- >>> ERROR: Access denied: chemical_id 11 expired on 2026-05-01

-- TEST 3: Attempt to double-book a piece of equipment.

-- Expected result: rejected by trg_prevent_reservation_overlap.

INSERT INTO equipment_reservation (equipment_id, person_id, start_time, end_time, purpose)

VALUES (9, 6, '2026-09-10 09:00', '2026-09-10 11:00', 'Sample prep')

        , (9, 12, '2026-09-10 10:00', '2026-09-10 12:00', 'Calibration');

-- >>> ERROR: Reservation overlap for equipment_id 9

-- TEST 4: Attempt to insert a chemical with a hazard class but no required certification.

-- Expected result: rejected by the CHECK constraint on chemical_inventory.

INSERT INTO chemical_inventory (name, hazard_class, quantity, unit, lab_id, storage_location, expiry_date, min_required_cert_id)

VALUES ('Benzene', 'carcinogen', 2.0, 'L', 3, 'Cabinet A2', '2027-01-01', NULL);

-- >>> ERROR: new row for relation "chemical_inventory" violates check constraint

-- TEST 5: Attempt to UPDATE an access_log row after the fact.

-- Expected result: rejected by trg_block_access_log_update — audit trail is immutable.

UPDATE access_log SET authorized = TRUE WHERE log_id = 205;

-- >>> ERROR: access_log is append-only; UPDATE is not permitted

-- TEST 6 (positive control): A certified radiation-safety officer checks out

-- a radioactive chemical — expected to SUCCEED.

INSERT INTO access_log (person_id, lab_id, resource_type, resource_id, access_type, authorized)

VALUES (3, 3, 'chemical', 8, 'checkout', TRUE);

-- >>> INSERT 0 1  (authorized recorded as TRUE)
5.2 Role-Based Access Control (RBAC) Definition
Role	Purpose	Privilege Summary
role_admin	System administrator	Full privileges on all tables
role_lab_manager	Manages a lab's inventory and certifications	Read/write on equipment, chemical_inventory, certification, personnel_certification; read-only elsewhere
role_faculty	PI / lab supervisor	Read on all research data; write on equipment_reservation; read-only on access_log for their own lab
role_student	Grad/undergrad researcher	Read on equipment, chemical_inventory (non-sensitive columns); write on equipment_reservation; insert-only on access_log
role_safety_officer	Compliance/safety auditing	Read-only on access_log, personnel_certification, chemical_inventory

-- ============================================================

-- ROLE CREATION

-- ============================================================

CREATE ROLE role_admin;

CREATE ROLE role_lab_manager;

CREATE ROLE role_faculty;

CREATE ROLE role_student;

CREATE ROLE role_safety_officer;

-- ============================================================

-- ADMIN — full control

-- ============================================================

GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO role_admin;

-- ============================================================

-- LAB MANAGER — manage inventory, equipment, certifications

-- ============================================================

GRANT SELECT ON department, personnel, lab, research_grant, equipment_reservation TO role_lab_manager;

GRANT SELECT, INSERT, UPDATE, DELETE ON equipment, chemical_inventory,

      certification, personnel_certification TO role_lab_manager;

GRANT INSERT ON access_log TO role_lab_manager;   -- append-only, enforced by trigger

-- ============================================================

-- FACULTY — oversight of their own grants/labs

-- ============================================================

GRANT SELECT ON department, lab, equipment, chemical_inventory,

      research_grant, certification, personnel_certification TO role_faculty;

GRANT SELECT, INSERT, UPDATE ON equipment_reservation TO role_faculty;

GRANT SELECT, INSERT ON access_log TO role_faculty;

-- ============================================================

-- STUDENT — day-to-day lab use

-- ============================================================

GRANT SELECT (equipment_id, equipment_name, category, lab_id, status) ON equipment TO role_student;

GRANT SELECT (chemical_id, name, hazard_class, lab_id, storage_location, expiry_date) ON chemical_inventory TO role_student;

GRANT SELECT, INSERT ON equipment_reservation TO role_student;

GRANT INSERT ON access_log TO role_student;        -- writes go through the safety triggers

GRANT SELECT ON v_active_certifications TO role_student; -- students can check their own cert status

-- ============================================================

-- SAFETY OFFICER — read-only audit access

-- ============================================================

GRANT SELECT ON access_log, personnel_certification, chemical_inventory,

      certification, v_chemicals_expiring_soon TO role_safety_officer;

-- ============================================================

-- REVOKES — explicitly close off dangerous defaults

-- ============================================================

REVOKE UPDATE, DELETE ON access_log FROM PUBLIC;   -- redundant with trigger, defense in depth

REVOKE ALL ON personnel_certification FROM role_student, role_faculty;

REVOKE ALL ON certification FROM role_student;

Design note: privileges are deliberately layered with the triggers in §4.1 as defense in depth — even a role that is mistakenly granted INSERT on access_log still cannot record an unauthorized hazardous-material access as authorized = TRUE, because the trigger evaluates certification status independently of which role issued the statement.

