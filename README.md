# Clinic Appointment and Diagnostics Platform — Database Design

A database design for a modern clinic that manages doctors, patients, appointments, consultations, diagnostic tests, reports, and payments in a clean and scalable way.

---

## Business Context

This is a focused clinic system — not a hospital-level giant. Key characteristics that shaped the design:

- **Appointment ≠ Consultation** — a patient books an appointment first; the consultation only happens if the appointment is attended. A no-show appointment has no consultation record.
- **Tests belong to consultations, not appointments** — a doctor prescribes tests during the actual consultation, not at the booking stage.
- **Reports link back to both the test order and the doctor** — so the doctor who prescribed the test can review the result.
- **One consultation → many tests** — a `test_orders` junction table handles this cleanly.
- **Prescriptions are separate from consultations** — medicines are tracked per consultation but in their own table to avoid stuffing.
- **Department is a separate entity** — doctors belong to departments (cardiology, dermatology, etc.).
- **Payment is linked to the appointment** — a single payment record covers the consultation fee; test payments can be tracked via `payment_for` field.

---

## Schema Overview

The database has **10 entities**:

| Entity | Purpose |
|---|---|
| `patients` | Patient demographics and contact info |
| `departments` | Clinic departments / specialties |
| `doctors` | Doctor details, specialization, consultation fee |
| `appointments` | Booking record — scheduled time, status, reason |
| `consultations` | Actual visit — diagnosis, symptoms, notes, follow-up |
| `prescriptions` | Medicines prescribed during a consultation |
| `diagnostic_tests` | Master list of available tests — name, category, price |
| `test_orders` | Tests prescribed for a patient during a consultation |
| `reports` | Results generated for a test order |
| `payments` | Payment per appointment — method, status, reference |

---

## Key Design Decisions

### 1. Appointment and Consultation are separate tables
An appointment is a booking — it may be pending, confirmed, cancelled, or a no-show. A consultation is the actual doctor-patient interaction. Separating them means cancelled appointments don't create ghost consultation records, and a patient's full visit history is queryable independently of whether they showed up.

### 2. Tests belong to `consultations`, not `appointments`
The doctor prescribes tests only after examining the patient. Linking test orders to `consultation_id` correctly reflects real clinic workflow.

### 3. `diagnostic_tests` is a master catalog table
Stores the list of all tests the clinic can perform (CBC, lipid panel, X-ray, etc.) with their price and normal ranges. `test_orders` is the junction that says "this test was ordered for this patient during this consultation." This avoids duplicating test metadata per patient.

### 4. `reports` link to `test_order_id`
A report is the result of a specific test ordered in a specific consultation. Linking to `test_order_id` preserves the full chain: patient → appointment → consultation → test order → report.

### 5. `prescriptions` as a separate table
A single consultation can result in multiple medicines. One row per medicine keeps the consultation record clean and makes it easy to query full medication history per patient.

### 6. `payment_for` field on payments
Values: `consultation`, `diagnostics`, `both` — allows one payment record to cover different billing scenarios without creating multiple payment tables.

---

## Clinic Workflow

```
Patient registers
      ↓
Books Appointment (with doctor)
      ↓
Appointment attended → Consultation created
      ↓
Doctor records diagnosis, symptoms, notes
      ↓
Doctor prescribes medicines → Prescriptions table
Doctor orders tests       → Test Orders table
      ↓
Lab generates results → Reports table
      ↓
Doctor reviews report
      ↓
Payment recorded → Payments table
```

---

## Relationships

```
departments        ||--o{   doctors             : has
patients           ||--o{   appointments        : books
doctors            ||--o{   appointments        : attends
appointments       ||--o|   consultations       : leads to
patients           ||--o{   consultations       : has
doctors            ||--o{   consultations       : conducts
consultations      ||--o{   prescriptions       : generates
consultations      ||--o{   test_orders         : prescribes
diagnostic_tests   ||--o{   test_orders         : ordered via
test_orders        ||--||   reports             : produces
patients           ||--o{   payments            : makes
appointments       ||--o{   payments            : billed via
```

---

## Entities and Attributes

### patients
| Column | Type | Notes |
|---|---|---|
| patient_id | int PK | |
| name | varchar | |
| phone | varchar | |
| email | varchar | |
| date_of_birth | date | Used to calculate age |
| gender | varchar | |
| blood_group | varchar | Critical for diagnostics |
| address | text | |
| created_at | datetime | |

### departments
| Column | Type | Notes |
|---|---|---|
| department_id | int PK | |
| name | varchar | Cardiology / Dermatology / General etc. |
| description | text | |

### doctors
| Column | Type | Notes |
|---|---|---|
| doctor_id | int PK | |
| name | varchar | |
| phone | varchar | |
| email | varchar | |
| specialization | varchar | More specific than department |
| qualification | varchar | MBBS / MD / MS etc. |
| department_id | int FK | → departments |
| consultation_fee | decimal | |
| is_active | boolean | |
| joined_at | datetime | |

### appointments
| Column | Type | Notes |
|---|---|---|
| appointment_id | int PK | |
| patient_id | int FK | → patients |
| doctor_id | int FK | → doctors |
| scheduled_at | datetime | |
| appointment_type | varchar | in_person / online |
| status | varchar | pending / confirmed / completed / cancelled / no_show |
| reason | text | Patient's stated reason for visit |
| booked_at | datetime | |

### consultations
| Column | Type | Notes |
|---|---|---|
| consultation_id | int PK | |
| appointment_id | int FK | → appointments (one-to-one) |
| patient_id | int FK | → patients |
| doctor_id | int FK | → doctors |
| consulted_at | datetime | |
| symptoms | text | |
| diagnosis | text | |
| notes | text | |
| follow_up_date | date | Next visit if needed |

### prescriptions
| Column | Type | Notes |
|---|---|---|
| prescription_id | int PK | |
| consultation_id | int FK | → consultations |
| medicine_name | varchar | |
| dosage | varchar | 500mg / 10ml etc. |
| frequency | varchar | twice daily / after meals etc. |
| duration_days | int | |
| instructions | text | |

### diagnostic_tests
| Column | Type | Notes |
|---|---|---|
| test_id | int PK | |
| name | varchar | CBC / Lipid Panel / X-Ray etc. |
| description | text | |
| category | varchar | blood / urine / imaging / biopsy |
| normal_range | varchar | Reference values |
| price | decimal | |

### test_orders
| Column | Type | Notes |
|---|---|---|
| test_order_id | int PK | |
| consultation_id | int FK | → consultations |
| patient_id | int FK | → patients |
| test_id | int FK | → diagnostic_tests |
| ordered_at | datetime | |
| status | varchar | pending / sample_collected / processing / completed |
| priority | varchar | routine / urgent |

### reports
| Column | Type | Notes |
|---|---|---|
| report_id | int PK | |
| test_order_id | int FK | → test_orders (one-to-one) |
| patient_id | int FK | → patients |
| doctor_id | int FK | → doctors (prescribing doctor) |
| generated_at | datetime | |
| result_value | text | Actual result data |
| result_status | varchar | normal / abnormal / critical |
| technician_notes | text | |
| reviewed_by_doctor | boolean | |
| report_url | varchar | PDF link of the report |

### payments
| Column | Type | Notes |
|---|---|---|
| payment_id | int PK | |
| patient_id | int FK | → patients |
| appointment_id | int FK | → appointments |
| payment_for | varchar | consultation / diagnostics / both |
| amount | decimal | |
| payment_method | varchar | cash / card / UPI / insurance |
| payment_status | varchar | pending / paid / failed / refunded |
| transaction_ref | varchar | |
| paid_at | datetime | |

---

## Tools Used

- **Eraser.io** — ERD diagram design

---
## ER Diagram

