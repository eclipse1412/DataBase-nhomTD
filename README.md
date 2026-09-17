# SafeLab

### University Research & Lab Asset Tracker

**A relational database design connecting research funding, laboratory assets, people, and access records.**

| Project | Details |
| --- | --- |
| Team | TD |
| Assignment | Project #4 |
| Focus | Database requirements and relational design |
| Scope | 10 core tables and 10 business rules |
| Current stage | Design documentation; SQL implementation pending |

## Overview

University laboratories need to connect their assets to the grants that fund them and to the people responsible for them. SafeLab models those relationships in a shared relational database.

The proposed system brings together departmental information, research grants, equipment, chemical inventory, certification history, equipment bookings, and access records. Its requirements emphasize traceability, valid certifications, consistent reservations, and an auditable record of access decisions.

This repository documents the design described in Team TD's report. It includes rewritten documentation and editable relationship diagrams. Database scripts, an application interface, and enforcement tests have not been supplied or implemented here.

## Planned capabilities

| Area | Intended behavior |
| --- | --- |
| Research funding | Associate funded assets with grants and their principal investigators. |
| Laboratory assets | Track equipment location, purchase value, and operational status. |
| Inventory records | Maintain quantities, storage references, expiry dates, and certification requirements. |
| Personnel certifications | Record certification renewals and evaluate validity at the time of a request. |
| Equipment reservations | Record bookings and prevent conflicting time intervals for the same asset. |
| Access auditing | Record permitted and denied requests while preventing application roles from changing historical log entries. |

These are requirements for a future implementation. The documentation itself does not enforce them.

## Database structure

| Table | Responsibility |
| --- | --- |
| `department` | Department names and building assignments |
| `personnel` | University members and their academic or staff roles |
| `lab` | Laboratory locations and departmental ownership |
| `research_grant` | Funding information, budgets, dates, and principal investigators |
| `certification` | Certification definitions and validity periods |
| `personnel_certification` | A person's certification history, including renewals |
| `equipment` | Equipment records, location, funding, value, and status |
| `chemical_inventory` | Inventory records and their required certification references |
| `equipment_reservation` | Equipment booking intervals and purposes |
| `access_log` | Access requests and authorization outcomes |

Most entities use a single identifier as their primary key. Certification history uses the composite key `(person_id, cert_id, issue_date)` so a person can receive the same certification on different dates.

The report discusses normalization through 3NF and BCNF.
## Business rules at a glance

- Equipment belongs to one lab; an asset's grant association is optional.
- A grant's principal investigator must have the `faculty` personnel role.
- Restricted inventory requires an appropriate, valid certification.
- Expired certifications must not authorize a request.
- A single equipment item must not have overlapping reservations.
- Purchases charged to a grant must stay within its approved budget.
- Expired inventory must not be checked out for use.
- Changes to certification and inventory records require designated database roles.
- Access records must preserve both allowed and denied outcomes and remain append-only for application roles.

## Implementation roadmap

- [x] Describe the project scope and core entities.
- [x] Document BR-1 through BR-10.
- [x] Provide the relational mapping, dictionary, and editable ER diagrams.
- [x] Record normalization reasoning and implementation questions.
- [ ] Resolve the open decisions in the design notes.
- [ ] Choose a database engine and implement the schema.
- [ ] Add constraints, transaction logic, privileges, and audit handling.
- [ ] Add synthetic sample data and representative queries.
- [ ] Verify authorization, budget limits, reservation conflicts, and audit retention.

## Team TD

- Trần Anh Hào - N24DECE068
- Chu Tự Đức - N24DECE063
- Nguyễn Anh Tuân - N24DECE101
