# Factory Management System (FMS)
## Functional Requirements Specification (FRS)
**Document Version:** 1.0.0-FINAL  
**Target Enterprise:** Medium-Sized Garment Manufacturing Facility (~100 Employees)  
**Jurisdiction:** Pakistan (Labor Laws, NADRA Standards, Provincial Statutes)  
**Document Status:** Baseline Functional Specification  
**Parent References:** [`FACTORY_MANAGEMENT_SYSTEM_BRD_SRS.md`](file:///e:/Projects/Factory%20Managemenny%20System/docs/FACTORY_MANAGEMENT_SYSTEM_BRD_SRS.md)  

---

## Document Overview & Engineering Scope

This Functional Requirements Specification (FRS) establishes the complete, production-ready functional contract for the Factory Management System (FMS). It translates high-level business goals and regional statutory requirements into explicit, testable system behaviors across 12 core functional modules.

### Industrial & Regional Context
* **Industry Sector:** Ready-Made Garments (RMG) / Apparel Manufacturing.
* **Workforce Scale:** Approximately 100 active personnel (spanning Cutting, Stitching, Finishing, Quality Control, Maintenance, Warehousing, and Management).
* **Compensation Modalities:** Hybrid payroll covering Monthly Fixed Salaried Staff, Daily-Wage Floor Workers, and Piece-Rate Stitching Operators.
* **National Compliance Baseline:** Pakistan Labor Laws (Factories Act 1934 / Provincial Enactments, EOBI Act 1976, Provincial Social Security SESSI/PESSI, Income Tax Ordinance 2001 Section 149, NADRA CNIC standards).

### Requirements & Assumptions Tagging Convention
* **`[CONFIRMED]`**: Validated enterprise and regulatory requirement.
* **`[ASSUMPTION - REQUIRING FACTORY CONFIRMATION]`**: Standard operational best practice marked for explicit factory executive sign-off prior to code implementation.

---

## Module 1: Authentication and User Management

### 1.1 Purpose & Main Features
Provides secure identity verification, credential lifecycle management, session governance, and granular Role-Based Access Control (RBAC) across five organizational tiers: **Super Admin**, **Factory Admin**, **Supervisor**, **Accountant**, and **Employee**.

* **Main Features:**
  - Secure credential login with email/username and password.
  - Role-Based Access Control (RBAC) and Attribute-Based Access Control (ABAC) for departmental/line data isolation.
  - Multi-Factor Authentication (MFA via TOTP) for administrative and financial accounts.
  - Brute-force protection, account lockout, and automated password reset workflows.
  - Session timeout and concurrent login prevention for privileged roles.

### 1.2 User Actions
* **Super Admin / Factory Admin:** Create user accounts, assign roles, bind users to employee profiles, force password resets, unlock locked accounts, deactivate users.
* **All Users:** Login, logout, initiate "Forgot Password", update password, configure TOTP MFA (if privileged).

### 1.3 Required Data Fields
| Field Name | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `user_id` | UUID | Mandatory | Unique system-wide user identifier (Primary Key). |
| `employee_id` | UUID | Optional | Foreign key link to `employees.id` (1-to-1 relationship). |
| `username` | VARCHAR(50) | Mandatory | Unique login handle (e.g., employee code or short name). |
| `email` | VARCHAR(120) | Optional | Work or personal email address (unique if provided). |
| `password_hash` | VARCHAR(255) | Mandatory | Argon2id or bcrypt cryptographically salted hash. |
| `role` | ENUM | Mandatory | `SUPER_ADMIN`, `FACTORY_ADMIN`, `SUPERVISOR`, `ACCOUNTANT`, `EMPLOYEE`. |
| `status` | ENUM | Mandatory | `ACTIVE`, `INACTIVE`, `LOCKED`, `PASSWORD_EXPIRED`. |
| `failed_login_count` | INTEGER | Mandatory | Counter for consecutive invalid authentication attempts. |
| `last_login_at` | TIMESTAMPTZ | Optional | Timestamp of most recent successful authentication. |
| `mfa_secret` | VARCHAR(128) | Optional | Encrypted TOTP secret key for 2FA. |
| `mfa_enabled` | BOOLEAN | Mandatory | Default `FALSE`; forced `TRUE` for Admin and Accountant. |

### 1.4 Validation Rules
1. Password must be $\ge 10$ characters, containing $\ge 1$ uppercase letter, $\ge 1$ lowercase letter, $\ge 1$ digit, and $\ge 1$ special character.
2. Account is automatically locked after 5 consecutive failed login attempts within a 15-minute window.
3. User cannot bind to an `employee_id` that is already bound to another active user.
4. Administrative sessions expire after 20 minutes of inactivity; employee self-service sessions expire after 60 minutes.

### 1.5 Permissions
* **Super Admin:** Full CRUD on all users, credentials, and role assignments.
* **Factory Admin:** Create/Edit Supervisor, Accountant, and Employee users within their factory.
* **Supervisor / Accountant / Employee:** No administrative access; change own password only.

### 1.6 Important Edge Cases
* **Terminated Employee Login:** If an employee's HR status changes to `TERMINATED` or `RESIGNED`, their linked system user account and all active JWT/session tokens must immediately revoke.
* **Supervisor Impersonation:** Prevent supervisors from altering passwords or email addresses of their peers or superiors.
* **Network Drop during MFA Setup:** If 2FA enrollment is interrupted before verification of the first code, MFA must not be activated.

### 1.7 Acceptance Criteria
* `GIVEN` an active user with valid credentials, `WHEN` they submit login with correct username and password, `THEN` the system issues a short-lived JWT session and redirects to the role-specific dashboard within 400ms.
* `GIVEN` an account subjected to 5 consecutive wrong passwords, `WHEN` the 5th attempt fails, `THEN` the account status transitions to `LOCKED` and subsequent attempts return HTTP `423 Locked`.

---

## Module 2: Employee Management

### 2.1 Purpose & Main Features
Maintains the master record of all factory personnel across their employment lifecycle, strictly tracking personal demographics, legal identification (NADRA CNIC), factory line assignments, and payment routing.

* **Main Features:**
  - Complete digital employee profile record (Personal, Contact, Emergency, Employment, Banking/Cash).
  - NADRA CNIC 13-digit format validation and expiry tracking.
  - Multi-tier employment classification: `PERMANENT`, `PROBATION`, `CONTRACTUAL`, `DAILY_WAGE`, `PIECE_RATE`.
  - Digital document attachment repository (CNIC scan, Contract/Appointment letter, Medical fitness certificate).
  - Employee status state machine (`DRAFT`, `ACTIVE`, `PROBATION`, `SUSPENDED`, `NOTICE_PERIOD`, `TERMINATED`, `RESIGNED`).

### 2.2 User Actions
* **Factory Admin:** Create employee profile, update personal/employment data, upload documents, transition status, initiate offboarding.
* **Accountant:** View employee directory, update bank/cash/mobile-wallet payment methods and tax status.
* **Supervisor:** View list and basic profiles of assigned subordinates (restricted from viewing compensation or banking).
* **Employee:** View own profile; submit emergency contact update requests.

### 2.3 Required Data Fields
| Field Name | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `id` | UUID | Mandatory | Primary Key. |
| `employee_code` | VARCHAR(30) | Mandatory | Unique plant badge ID (e.g., `GAR-0104`). |
| `biometric_device_user_id` | VARCHAR(30) | Optional | Numerical ID registered on ZKTeco/biometric clocks. |
| `first_name` | VARCHAR(60) | Mandatory | Employee first name. |
| `last_name` | VARCHAR(60) | Mandatory | Employee surname/father's name. |
| `cnic_number` | VARCHAR(15) | Mandatory | NADRA CNIC: `XXXXX-XXXXXXX-X` format (Unique, Encrypted). |
| `cnic_expiry_date` | DATE | Mandatory | Expiry date printed on the physical CNIC card. |
| `date_of_birth` | DATE | Mandatory | Date of birth (Must validate $\ge 18$ years old). |
| `gender` | ENUM | Mandatory | `MALE`, `FEMALE`, `TRANSGENDER`. |
| `contact_phone` | VARCHAR(20) | Mandatory | Mobile phone number (e.g., `0300-1234567`). |
| `emergency_contact_phone`| VARCHAR(20) | Mandatory | Emergency contact phone number and relationship. |
| `employment_type` | ENUM | Mandatory | `PERMANENT`, `PROBATION`, `CONTRACT`, `DAILY_WAGE`, `PIECE_RATE`. |
| `status` | ENUM | Mandatory | Lifecycle status (Active, Probation, Terminated, etc.). |
| `hire_date` | DATE | Mandatory | Official date of joining the factory. |
| `department_id` | UUID | Mandatory | FK to `departments.id`. |
| `designation_id` | UUID | Mandatory | FK to `designations.id`. |
| `supervisor_id` | UUID | Optional | FK to `employees.id` (Reporting line lead). |
| `payment_mode` | ENUM | Mandatory | `CASH`, `BANK_TRANSFER`, `EASYPAISA`, `JAZZCASH`. |
| `bank_iban` | VARCHAR(24) | Optional | 24-character Pakistani IBAN (if `BANK_TRANSFER`). |
| `mobile_wallet_number` | VARCHAR(15) | Optional | Mobile wallet number (if `EASYPAISA`/`JAZZCASH`). |
| `eobi_number` | VARCHAR(30) | Optional | EOBI individual registration number. |
| `sessi_pessi_number` | VARCHAR(30) | Optional | Provincial social security card registration number. |

### 2.4 Validation Rules
1. **CNIC Validation:** Must match regex `^[0-9]{5}-[0-9]{7}-[0-9]{1}$`. The last digit parity must align with gender: Odd for Male, Even for Female.
2. **Age Compliance:** Worker must be $\ge 18$ years of age at `hire_date` under Pakistan Factories Act rules against child labor.
3. **IBAN Validation:** If `payment_mode = BANK_TRANSFER`, IBAN must start with `PK` and contain exactly 24 alphanumeric characters.
4. **Duplicate Prevention:** `employee_code`, `cnic_number`, and `biometric_device_user_id` must be globally unique across active and archived records.

### 2.5 Permissions
* **Factory Admin:** Full CRUD on profiles.
* **Accountant:** Read all; Edit financial/banking/statutory fields.
* **Supervisor:** Read-only for subordinate profiles (excluding salary and bank info).
* **Employee:** Read-only for own profile.

### 2.6 Important Edge Cases
* **Worker Without CNIC:** Helpers or seasonal stitchers presenting only a NADRA token/receipt without a finalized CNIC card. *System Rule:* System allows temporary onboarding with a mandatory 60-day expiry grace period flag `cnic_pending_token = TRUE`.
* **CNIC Expiry during Service:** System generates an HR alert 30 days prior to CNIC expiry.
* **Re-hiring Terminated Worker:** System detects matching CNIC and prompts HR to un-archive past profile with service history rather than creating a duplicate record.

### 2.7 Acceptance Criteria
* `GIVEN` an onboarding form with an invalid CNIC `12345-1234567`, `WHEN` HR attempts submission, `THEN` the system rejects with `Invalid CNIC format. Expected 13 digits: XXXXX-XXXXXXX-X`.
* `GIVEN` a date of birth indicating age 16, `WHEN` submitted, `THEN` the system rejects with `Age Violation: Worker must be at least 18 years old`.

---

## Module 3: Department and Designation Management

### 3.1 Purpose & Main Features
Defines the functional structure and operational hierarchy of the garment factory. Groups production operations into standardized divisions and establishes job titles, skill grading, and cost center associations.

* **Main Features:**
  - Standard Garment Divisions: Cutting, Stitching Lines (Lines 1 to N), Trimming & Washing, Ironing & Pressing, Quality Checking (QC), Packing & Finished Goods, Maintenance & Mechanics, Stores & Fabric Warehouse, HR & Accounts.
  - Designation and Grade Band Setup (e.g., Helper Grade A, Junior Stitcher, Senior Overlock Master, Cutting Master, Line Supervisor, QC Auditor).
  - Cost Center mapping for labor cost accounting.
  - Organizational hierarchy tree with recursive reporting lines and Directed Acyclic Graph (DAG) cycle prevention.

### 3.2 User Actions
* **Factory Admin:** Add new department, edit department name/code, assign department head, create designations, define wage scale bands ($Min - Max$).
* **Supervisor / Accountant:** View department listings, view designations and headcount tallies.

### 3.3 Required Data Fields
| Field Name | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `department_id` | UUID | Mandatory | Primary Key. |
| `dept_code` | VARCHAR(20) | Mandatory | Unique department code (e.g., `DEPT-STITCH`). |
| `dept_name` | VARCHAR(80) | Mandatory | Department name (e.g., Stitching Floor - Line 1). |
| `cost_center_code` | VARCHAR(30) | Mandatory | Accounting cost allocation code. |
| `parent_dept_id` | UUID | Optional | Parent department for sub-lines (Self-referencing FK). |
| `head_employee_id` | UUID | Optional | Department manager/lead (FK to `employees.id`). |
| `is_production_unit` | BOOLEAN | Mandatory | Flag indicating direct garment production floor line. |
| `designation_id` | UUID | Mandatory | Primary Key for designations. |
| `designation_code` | VARCHAR(20) | Mandatory | Unique code (e.g., `DESIG-STITCH-SR`). |
| `designation_title` | VARCHAR(80) | Mandatory | Title (e.g., Senior Flatlock Operator). |
| `skill_grade` | ENUM | Mandatory | `UNSKILLED`, `SEMI_SKILLED`, `SKILLED`, `HIGHLY_SKILLED`, `SUPERVISORY`, `EXECUTIVE`. |
| `min_base_salary` | DECIMAL(10,2)| Mandatory | Minimum allowed base wage for this designation (PKR). |
| `max_base_salary` | DECIMAL(10,2)| Mandatory | Maximum allowed base wage for this designation (PKR). |

### 3.4 Validation Rules
1. `dept_code` and `designation_code` must be alphanumeric, uppercase, and unique.
2. `min_base_salary` cannot fall below the legally mandated provincial minimum wage gazette.
3. A department cannot be assigned as its own parent or a descendant of its own children (Cycle detection check).
4. A department or designation cannot be deleted if active employees are currently assigned to it.

### 3.5 Permissions
* **Factory Admin:** Full CRUD.
* **Accountant:** Read-only (access to Cost Centers and Designations).
* **Supervisor / Employee:** Read-only (organization directory).

### 3.6 Important Edge Cases
* **Splitting or Merging Stitching Lines:** Line 1 and Line 2 merged for a specialized production order. System allows bulk-reassignment of employees from one department code to another in a single transaction without corrupting historical attendance records.

### 3.7 Acceptance Criteria
* `GIVEN` an existing Department A, `WHEN` an admin sets its parent to Department B, which already has parent Department A, `THEN` the system detects a circular dependency and rejects the transaction.
* `GIVEN` a designation with 12 assigned stitchers, `WHEN` an admin clicks "Delete", `THEN` the system blocks the action with `Cannot delete designation with active employees`.

---

## Module 4: Attendance Management

### 4.1 Purpose & Main Features
Governs daily workforce time-tracking across physical biometric clock inputs, handles industrial grace periods, enforces late penalties, accommodates special Friday prayer schedules, and provides dual-approval manual regularization.

* **Main Features:**
  - Automated ingestion of biometric punches (ZKTeco/LAN push or USB/desktop sync).
  - Anti-passback debounce filtering (ignoring duplicate taps within 3 minutes).
  - Automatic classification: `PRESENT`, `LATE`, `HALF_DAY`, `ABSENT`, `ON_LEAVE`, `GAZETTED_HOLIDAY`.
  - Friday (*Jumma*) Extended Prayer break handling (1.5–2 hours) vs. standard weekday breaks.
  - Missing check-out detection and supervisor-initiated regularization with mandatory reason codes.
  - Factory Admin dual-approval for all manual punch corrections.
  - End-of-month attendance freeze locking records against retroactive modifications.

### 4.2 User Actions
* **Supervisor:** View daily floor attendance roster, flag missing check-outs, submit punch correction requests for line workers with explanations.
* **Factory Admin:** Review and approve/reject supervisor attendance regularizations; execute monthly attendance freeze.
* **Accountant:** Export verified monthly attendance timesheets into the payroll calculation engine.
* **Employee:** View individual punch times and daily attendance status via mobile or supervisor kiosk.

### 4.3 Required Data Fields
| Field Name | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `attendance_id` | UUID | Mandatory | Primary Key. |
| `employee_id` | UUID | Mandatory | FK to `employees.id`. |
| `work_date` | DATE | Mandatory | Operational date (YYYY-MM-DD). |
| `shift_id` | UUID | Mandatory | FK to `shifts.id` (Rostered shift). |
| `check_in_time` | TIMESTAMPTZ | Optional | Actual first clock-in timestamp. |
| `check_out_time` | TIMESTAMPTZ | Optional | Actual last clock-out timestamp. |
| `raw_minutes` | INTEGER | Mandatory | Total raw elapsed minutes between in and out. |
| `net_work_hours` | DECIMAL(5,2)| Mandatory | Payable work hours (net of break durations). |
| `late_in_minutes` | INTEGER | Mandatory | Minutes arrived after shift start. |
| `early_exit_minutes`| INTEGER | Mandatory | Minutes departed before shift end. |
| `status` | ENUM | Mandatory | `PRESENT`, `LATE`, `HALF_DAY`, `ABSENT`, `ON_LEAVE`, `HOLIDAY`, `GATE_LOCKED`. |
| `is_manual_override`| BOOLEAN | Mandatory | Default `FALSE`; set `TRUE` on regularization. |
| `override_by_id` | UUID | Optional | FK to `employees.id` (Supervisor who requested). |
| `override_approved_by`| UUID | Optional | FK to `employees.id` (Factory Admin who approved). |
| `override_reason` | TEXT | Optional | Mandatory explanation text for manual correction. |
| `is_locked` | BOOLEAN | Mandatory | Default `FALSE`; permanently `TRUE` once payroll closes. |

### 4.4 Validation Rules
1. **Debounce Filter:** Successive punch logs for the same `biometric_device_user_id` within 180 seconds are discarded as biometric bounces.
2. **Late-In Grace Period:** Arrival $\le 15$ minutes after shift start = `PRESENT` (no salary cut). Arrival $> 15$ minutes = `LATE`.
3. *[ASSUMPTION - REQUIRING FACTORY CONFIRMATION]*: **3-Late Penalty Rule:** Accumulation of 3 `LATE` marks in a calendar month results in an automatic deduction of 0.5 payable days from the monthly total.
4. **Half-Day Minimum:** If net payable hours are $\ge 4.0\text{ hours}$ but $< 7.0\text{ hours}$, status evaluates to `HALF_DAY`. Net hours $< 4.0\text{ hours}$ without approved leave evaluate to `ABSENT`.
5. **No Retroactive Edits:** Once `is_locked = TRUE`, no user (including Factory Admin) can update attendance records.

### 4.5 Permissions
* **Factory Admin:** Full viewing; Final approval for manual overrides; Lock monthly attendance.
* **Supervisor:** View subordinate attendance; Submit punch regularization requests.
* **Accountant:** Read-only access to final attendance registers.
* **Employee:** View own attendance records only.

### 4.6 Important Edge Cases
* **Missing Check-Out:** Worker clocks in at 08:00, works full shift, but forgets to punch out at 17:00. *System Behavior:* System assigns status `MISSING_CHECKOUT` and credits 0 hours until the supervisor submits a regularization confirming the exit time, approved by the Factory Admin.
* **Friday Prayer Overstay:** Worker returns 45 minutes late from Jumma prayer. The system calculates additional break time and logs an early exit / partial hour deduction if total hours fall short.
* **Power Outage During Shift Change:** Standalone device stores punches on internal memory; when LAN reconnects, batch sync updates attendance timestamps without overwriting existing valid punches.

### 4.7 Acceptance Criteria
* `GIVEN` an employee scheduled for 08:00 to 17:00, `WHEN` they clock in at 08:12, `THEN` the record status is `PRESENT` and `late_in_minutes = 12`.
* `GIVEN` an employee clocking in at 08:25, `WHEN` evaluated, `THEN` the status is `LATE` and `late_in_minutes = 25`.
* `GIVEN` a locked attendance period, `WHEN` an edit request is sent via API, `THEN` the system returns HTTP `403 Forbidden` with `Attendance period is permanently locked`.

---

## Module 5: Shift Management

### 5.1 Purpose & Main Features
Configures and schedules operational shift patterns across the garment plant floor, supporting standard day shifts, overnight shifts crossing midnight, rotational scheduling, and seasonal Ramadan timing adjustments.

* **Main Features:**
  - Standard Shift Configuration (Morning Shift, Evening Shift, Night Shift).
  - Overnight shift logic crossing midnight (`is_overnight = TRUE`) with correct date anchoring.
  - Day-specific break schedules (accommodating Friday Jumma prayers).
  - Ramadan Working Hours schedule template (e.g., shortened 6-hour shifts without lunch break).
  - Weekly roster scheduling per department, production line, or individual worker.

### 5.2 User Actions
* **Factory Admin:** Create/Edit shift templates, define grace periods, activate Ramadan calendar, configure factory weekly off days (usually Sunday).
* **Supervisor:** Assign line workers to shifts, manage weekly rotation rosters, view upcoming line coverage.
* **Employee:** View own upcoming shift schedule.

### 5.3 Required Data Fields
| Field Name | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `shift_id` | UUID | Mandatory | Primary Key. |
| `shift_code` | VARCHAR(20) | Mandatory | Unique shift code (e.g., `SHIFT-GEN`, `SHIFT-NIGHT`). |
| `shift_name` | VARCHAR(60) | Mandatory | Name (e.g., General Garment Day Shift). |
| `start_time` | TIME | Mandatory | Scheduled start time (e.g., `08:00:00`). |
| `end_time` | TIME | Mandatory | Scheduled end time (e.g., `17:00:00`). |
| `is_overnight` | BOOLEAN | Mandatory | Default `FALSE`; `TRUE` if shift ends on next calendar date. |
| `late_grace_minutes`| INTEGER | Mandatory | Grace period before late mark (e.g., 15 minutes). |
| `standard_break_minutes`| INTEGER | Mandatory | Lunch/tea break duration Monday–Thursday & Saturday (e.g., 60 mins). |
| `friday_break_minutes`| INTEGER | Mandatory | Extended Jumma prayer break on Fridays (e.g., 90–120 mins). |
| `half_day_hours` | DECIMAL(4,2)| Mandatory | Minimum hours required for half-day credit (e.g., 4.0 hours). |
| `full_day_hours` | DECIMAL(4,2)| Mandatory | Minimum hours required for full-day credit (e.g., 8.0 hours). |
| `is_active` | BOOLEAN | Mandatory | Default `TRUE`. |

### 5.4 Validation Rules
1. `start_time` and `end_time` cannot be identical.
2. If `is_overnight = FALSE`, `end_time` must be chronologically after `start_time`.
3. Total shift duration minus `standard_break_minutes` cannot evaluate to $< 4.0\text{ hours}$.
4. A worker cannot be assigned to two overlapping shifts on the same calendar date.

### 5.5 Permissions
* **Factory Admin:** Full CRUD on shift templates and Ramadan overrides.
* **Supervisor:** Assign employees to predefined shifts; view shift rosters.
* **Accountant / Employee:** Read-only view.

### 5.6 Important Edge Cases
* **Midnight Crossing Shift (Night Shift: 20:00 to 05:00):** Worker punches in at 19:55 on Monday and punches out at 05:05 on Tuesday morning. *System Rule:* Both punches are anchored to Monday’s operational timesheet. Tuesday remains free for the next scheduled shift.
* **Ramadan Gazette Transition:** When the Admin switches the factory calendar to "Ramadan Mode", the system automatically compresses the shift schedule to statutory working hours without altering base monthly wage structures.

### 5.7 Acceptance Criteria
* `GIVEN` an overnight shift starting at 21:00 and ending at 06:00, `WHEN` an employee clocks in at 20:50 and clocks out at 06:10 next morning, `THEN` the system records a single attendance entry for the starting date with 8.0 payable hours (net of break).
* `GIVEN` an employee scheduled on Shift A, `WHEN` a supervisor attempts to assign them to Shift B on the same day with overlapping hours, `THEN` the system rejects with `Roster Conflict: Overlapping shift assignment`.

---

## Module 6: Leave Management

### 6.1 Purpose & Main Features
Governs employee leave requests, tracks annual statutory quotas in accordance with the Pakistan Factories Act 1934, provides multi-level approval workflows, and automatically feeds unpaid leaves into the payroll engine.

* **Main Features:**
  - Statutory Leave Quotas under Pakistan Labor Laws:
    - **Annual (Earned) Leave:** 14 consecutive calendar days after 12 months continuous service.
    - **Casual Leave:** 10 days fully paid per calendar year.
    - **Sick / Medical Leave:** 16 days at half pay (or 10 days full pay per factory policy).
    - **Festival Holidays:** 10–13 gazetted public holidays with full pay.
  - Multi-tier approval: Worker applies $\rightarrow$ Supervisor recommends $\rightarrow$ Factory Admin finalizes.
  - Real-time leave balance ledger (Quota, Used, Pending, Remaining Balance).
  - Configurable "Sandwich Rule" toggle (counting intervening holidays/weekends as leave).
  - Integration with Attendance Engine (auto-marking `ON_LEAVE` and preventing false absent marks).

### 6.2 User Actions
* **Employee:** View leave balance, submit leave application with reason and optional medical attachment, cancel pending application.
* **Supervisor:** View team leave calendar, review and approve/reject subordinate leave applications with remarks.
* **Factory Admin:** Final approval for extended or medical leaves; adjust leave balances; declare annual gazetted holidays.
* **Accountant:** View unpaid leaves / Loss of Pay (LOP) summaries for salary deductions.

### 6.3 Required Data Fields
| Field Name | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `leave_application_id`| UUID | Mandatory | Primary Key. |
| `employee_id` | UUID | Mandatory | FK to `employees.id`. |
| `leave_type` | ENUM | Mandatory | `CASUAL`, `ANNUAL_EARNED`, `SICK`, `MATERNITY`, `UNPAID_LOP`. |
| `start_date` | DATE | Mandatory | Leave start date. |
| `end_date` | DATE | Mandatory | Leave end date. |
| `total_leave_days` | DECIMAL(4,1)| Mandatory | Total days (e.g., 1.0, 0.5 for half-day, 3.0). |
| `reason` | TEXT | Mandatory | Stated reason for leave. |
| `medical_doc_url` | VARCHAR(255)| Optional | URL/path to uploaded medical certificate. |
| `status` | ENUM | Mandatory | `PENDING_SUPERVISOR`, `APPROVED_SUPERVISOR`, `APPROVED_FINAL`, `REJECTED`, `CANCELLED`. |
| `supervisor_id` | UUID | Optional | FK to `employees.id` (First-tier reviewer). |
| `supervisor_remarks`| TEXT | Optional | Supervisor comments on recommendation/rejection. |
| `final_approved_by` | UUID | Optional | FK to `employees.id` (Factory Admin). |

### 6.4 Validation Rules
1. `end_date` must be $\ge `start_date`$.
2. Paid leave cannot be applied if `total_leave_days` exceeds the employee’s current remaining balance in that leave category, unless submitted as `UNPAID_LOP`.
3. Sick leave $> 2$ consecutive days mandates uploading a medical certificate document.
4. *[ASSUMPTION - REQUIRING FACTORY CONFIRMATION]*: **Probationer Restriction:** Workers in their first 90 days of probation cannot take paid Annual or Casual leave.
5. An approved leave cannot be cancelled if the leave date falls in a locked payroll period.

### 6.5 Permissions
* **Employee:** Apply for own leave; cancel own pending leaves.
* **Supervisor:** Recommend/Reject leave for direct subordinates.
* **Factory Admin:** Final approval/override for all leaves; manage annual leave quotas.
* **Accountant:** Read-only access to leave balances and LOP reports.

### 6.6 Important Edge Cases
* **The "Sandwich Rule" Scenario:** Employee takes leave on Saturday and Monday, with Sunday being the weekly rest day. *System Rule:* If the "Sandwich Rule" is enabled in system settings, Sunday is deducted as a paid leave day (Total = 3 days). If disabled, Sunday remains a rest day (Total = 2 days).
* **Mass Eid Leave Applications:** 50 stitchers apply for leave surrounding Eid holidays. The system flags a "Production Line Deficit Warning" to the Factory Admin if approved leaves exceed 20% of a stitching line's operational capacity.

### 6.7 Acceptance Criteria
* `GIVEN` an employee with 3 days Casual Leave remaining, `WHEN` they apply for 4 days Casual Leave, `THEN` the system blocks the application with `Insufficient Casual Leave balance (Available: 3, Requested: 4)`.
* `GIVEN` an approved leave application, `WHEN` the attendance engine runs for that date, `THEN` the employee is automatically recorded as `ON_LEAVE` and excluded from the daily Absenteeism report.

---

## Module 7: Salary, Advance (Peshgi) and Payroll

### 7.1 Purpose & Main Features
Calculates complete, auditable monthly compensation across fixed, daily-wage, and piece-rate workers. Incorporates overtime wages, manages salary advance (*Peshgi*) recovery ledgers, calculates statutory deductions (EOBI, SESSI/PESSI, FBR tax), and generates cash denomination breakdowns and bank disbursement files.

* **Main Features:**
  - Hybrid Compensation Calculations:
    - **Monthly Fixed Salaried:** Prorated based on $(\text{Calendar Days} - \text{LOP Days} - \text{Unexcused Absent Days})$.
    - **Daily-Wage Workers:** Calculated based on exact days present + approved paid holidays.
    - **Piece-Rate Stitchers:** Calculated based on verified completed bundle units $\times$ unit rate.
  - **Advance Salary (*Peshgi*) Ledger:** Multi-month loan disbursement, monthly installment deduction, and remaining balance tracking.
  - Statutory Deductions Engine (Configurable for Pakistan Labor Laws):
    - **EOBI Pension:** Employee deduction (1% of statutory minimum wage) + Employer contribution (5%).
    - **Provincial Social Security (SESSI / PESSI):** Employer contribution (e.g., 6% up to wage ceiling).
    - **FBR Salary Tax (Section 149):** Progressive monthly tax withholding based on annual income estimate.
  - **Cash Denomination Sheet Generator:** Calculates exact counts of Pakistani Rupee notes (`PKR 5000, 1000, 500, 100, 50, 20, 10`) required for cash payroll withdrawal from the bank.
  - Dual-phase payroll: Pre-run Draft $\rightarrow$ Reconcile $\rightarrow$ Factory Admin Approval $\rightarrow$ Permanent Payroll Lock.

### 7.2 User Actions
* **Accountant:** Create payroll batch, trigger calculation, review draft register, enter ad-hoc bonuses/penalties, generate bank EFT files or Cash Denomination Sheets, release payslips.
* **Factory Admin:** Review draft payroll, audit overtime and deduction totals, perform final electronic approval and batch freeze.
* **Employee:** View and download itemized monthly payslip (or receive printed envelope slip).

### 7.3 Required Data Fields
| Field Name | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `payslip_id` | UUID | Mandatory | Primary Key. |
| `payroll_batch_id` | UUID | Mandatory | FK to `payroll_batches.id`. |
| `employee_id` | UUID | Mandatory | FK to `employees.id`. |
| `wage_type` | ENUM | Mandatory | `MONTHLY_FIXED`, `DAILY_WAGE`, `PIECE_RATE`. |
| `payable_days` | DECIMAL(4,1)| Mandatory | Net days eligible for salary payment. |
| `base_salary_earned`| DECIMAL(10,2)| Mandatory | Earned basic pay before allowances (PKR). |
| `allowances_total` | DECIMAL(10,2)| Mandatory | Conveyance, attendance bonus, medical allowances (PKR). |
| `overtime_pay` | DECIMAL(10,2)| Mandatory | Earned overtime pay (from Overtime Module). |
| `piece_rate_earnings`| DECIMAL(10,2)| Mandatory | Earnings from stitched garment units (if applicable). |
| `gross_pay` | DECIMAL(10,2)| Mandatory | Total earned: $\text{Base} + \text{Allowances} + \text{OT} + \text{Piece-Rate}$. |
| `peshgi_deduction` | DECIMAL(10,2)| Mandatory | Scheduled monthly salary advance recovery installment. |
| `eobi_deduction` | DECIMAL(10,2)| Mandatory | Employee share of EOBI (PKR). |
| `income_tax_deduction`| DECIMAL(10,2)| Mandatory | FBR Section 149 tax withholding (PKR). |
| `other_deductions` | DECIMAL(10,2)| Mandatory | Disciplinary penalties, damage deductions, canteen. |
| `total_deductions` | DECIMAL(10,2)| Mandatory | Sum of all deductions. |
| `net_pay` | DECIMAL(10,2)| Mandatory | Final payable: $\text{Gross Pay} - \text{Total Deductions}$. |
| `payment_mode` | ENUM | Mandatory | `CASH`, `BANK_TRANSFER`, `EASYPAISA`, `JAZZCASH`. |
| `payment_status` | ENUM | Mandatory | `UNPAID`, `PAID`, `HELD`. |

### 7.4 Validation Rules
1. **Mathematical Invariant:** $\text{Net Pay} = \text{Gross Pay} - \text{Total Deductions}$ must hold strictly to 2 decimal places.
2. **Non-Negative Pay Guardrail:** Net pay cannot be negative. If scheduled loan/advance deductions exceed 50% of gross pay, the system caps the advance deduction at 50% and rolls over the remaining loan balance to the next month.
3. **Provincial Minimum Wage Warning:** If a full-time permanent worker’s gross wage evaluates below the notified Provincial Minimum Wage, the system flags a "Statutory Non-Compliance Warning" on the draft register.
4. **Permanent Freezing:** Once the Factory Admin approves the payroll batch, all associated records, attendance, and advance deductions become immutable.

### 7.5 Permissions
* **Accountant:** Create, calculate, edit draft ad-hoc items, generate bank/cash exports.
* **Factory Admin:** View drafts, approve batch, execute lock.
* **Supervisor:** No access to salary or payroll data.
* **Employee:** View own finalized payslip only.

### 7.6 Important Edge Cases
* **Mid-Month Advance (*Peshgi*) Granted After Draft Run:** Worker receives PKR 5,000 emergency cash advance after the accountant runs the preliminary draft. *System Rule:* Re-running the batch recalculates net pay and updates the advance ledger automatically prior to final lock.
* **Cash Currency Rounding:** Net cash pay with fractional Rupee decimals (e.g., PKR 24,532.50) is rounded to the nearest integer (PKR 24,533) with the fractional difference carried in a factory rounding ledger.

### 7.7 Acceptance Criteria
* `GIVEN` a worker with Gross Pay = PKR 35,000 and Advance Loan Installment = PKR 5,000, EOBI = PKR 320, `WHEN` payroll is computed, `THEN` Net Pay evaluates exactly to $\text{PKR } 29,680.00$.
* `GIVEN` total factory cash payroll of PKR 1,845,000 across 60 cash workers, `WHEN` the Accountant clicks "Generate Denomination Sheet", `THEN` the system outputs the exact count of 5000, 1000, 500, 100, 50, 20, and 10 Rupee notes whose sum equals PKR 1,845,000.

---

## Module 8: Overtime Management

### 8.1 Purpose & Main Features
Manages the requisition, tracking, calculation, and authorization of hours worked beyond standard shift limits, adhering strictly to Pakistan labor statutory overtime multipliers.

* **Main Features:**
  - Automated detection of overtime hours from biometric check-out timestamps.
  - Statutory Multiplier Calculation under Section 47 of the Factories Act 1934:
    $$\text{Overtime Rate} = 2.0 \times \text{Ordinary Hourly Wage}$$
    $$\text{Ordinary Hourly Wage} = \frac{\text{Basic Salary} + \text{Permanent Allowances}}{\text{Standard Monthly Working Hours (e.g., 208 hrs)}}$$
  - Supervisor pre-requisition or post-shift validation workflow.
  - Safety thresholds: Maximum daily overtime limits (e.g., max 2 to 3 hours/day to prevent worker fatigue).
  - Integration with the Monthly Payroll Engine.

### 8.2 User Actions
* **Supervisor:** Submit overtime requisition for production line, verify and approve actual OT hours worked by line workers.
* **Factory Admin:** Audit factory-wide overtime hours, approve exceptional overtime requests exceeding daily limits.
* **Accountant:** Review verified overtime hours and rate calculations prior to payroll calculation.
* **Employee:** View logged overtime hours on daily timesheet.

### 8.3 Required Data Fields
| Field Name | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `overtime_record_id`| UUID | Mandatory | Primary Key. |
| `employee_id` | UUID | Mandatory | FK to `employees.id`. |
| `work_date` | DATE | Mandatory | Date overtime was performed. |
| `raw_ot_minutes` | INTEGER | Mandatory | Raw elapsed minutes beyond shift end. |
| `approved_ot_hours` | DECIMAL(4,2)| Mandatory | Hours approved by supervisor (rounded to 0.5h blocks). |
| `hourly_rate` | DECIMAL(8,2)| Mandatory | Base hourly wage calculated from employee master. |
| `ot_multiplier` | DECIMAL(3,2)| Mandatory | Statutory multiplier (Standard: `2.00`). |
| `total_ot_amount` | DECIMAL(10,2)| Mandatory | Earned OT: $\text{Hours} \times (\text{Hourly Rate} \times \text{Multiplier})$. |
| `supervisor_id` | UUID | Mandatory | FK to `employees.id` (Approving supervisor). |
| `approval_status` | ENUM | Mandatory | `PENDING`, `APPROVED`, `REJECTED`. |
| `is_payroll_processed`| BOOLEAN | Mandatory | Default `FALSE`; set `TRUE` when batch is locked. |

### 8.4 Validation Rules
1. Raw overtime $< 30\text{ minutes}$ is considered incidental and disregarded unless explicitly authorized.
2. Approved overtime must round to the nearest 0.5-hour increment (e.g., 65 mins = 1.0 hour, 95 mins = 1.5 hours).
3. Overtime hours on gazetted national holidays or weekly rest days evaluate at the designated holiday rate ($2.0\times$ statutory).
4. *[ASSUMPTION - REQUIRING FACTORY CONFIRMATION]*: **Maximum Overtime Cap:** No employee may perform $> 12\text{ hours}$ of overtime in a single week to maintain compliance with industrial labor safety standards.

### 8.5 Permissions
* **Supervisor:** Propose and verify OT for assigned subordinates.
* **Factory Admin:** Full visibility; Final approval on exceptional overtime $> 3\text{ hrs/day}$.
* **Accountant:** Read-only access to approved OT registers.
* **Employee:** View own overtime hours.

### 8.6 Important Edge Cases
* **Worker Stays in Canteen / Factory without Working:** Biometric check-out shows exit at 20:00 (3 hours past shift end), but supervisor did not authorize overtime. *System Rule:* The system flags the record as `UNAUTHORIZED_OVERTIME`; payable OT hours remain `0.00` until the supervisor explicitly approves the justification.

### 8.7 Acceptance Criteria
* `GIVEN` an employee with an ordinary hourly wage of PKR 150/hr, `WHEN` they complete 2.0 approved overtime hours on a standard working day, `THEN` the system calculates the earned OT as $2.0 \times (150 \times 2.0) = \text{PKR } 600.00$.
* `GIVEN` an unapproved biometric exit 90 minutes late, `WHEN` payroll is computed without supervisor sign-off, `THEN` overtime hours are excluded from payment.

---

## Module 9: Role-Based Dashboards

### 9.1 Purpose & Main Features
Provides real-time operational and financial visibility tailored to the specific persona logging in. Ensures shop-floor supervisors see live staffing coverage, factory admins see high-level plant KPIs and cost trends, and employees see individual records.

* **Main Features:**
  - **Factory Admin Command Center:** Real-time floor headcount, today’s absenteeism %, active stitching line coverage, MTD overtime cost run-rate, pending approvals queue, and biometric terminal connectivity status.
  - **Supervisor Shop-Floor Board:** Live line staffing status (Present vs. Expected operators), late-in alerts, missing punch exceptions, and pending team leave requests.
  - **Accountant Financial View:** Payroll cycle countdown, pending advances (*Peshgi*) awaiting disbursement, projected monthly wage liability, and tax/statutory deduction summaries.
  - **Employee Self-Service View:** Current month attendance calendar, leave balance ledger, recent punch logs, and latest payslip download.

### 9.2 User Actions
* **All Roles:** View assigned role-specific dashboard upon login, filter views by date or department (where permitted), click action cards to navigate to pending approvals.

### 9.3 Required Data Fields (Aggregated Metrics)
* *Plant Operations:* `total_active_workforce`, `present_today_count`, `absent_today_count`, `late_today_count`, `on_leave_today_count`.
* *Line Coverage:* `department_id`, `required_operators`, `actual_operators_present`, `coverage_percentage`.
* *Financial Run-rate:* `mtd_overtime_hours`, `estimated_mtd_gross_payroll`, `pending_advance_requests_count`.
* *Hardware Status:* `terminal_id`, `terminal_name`, `ip_address`, `status (ONLINE/OFFLINE)`, `last_ping_at`.

### 9.4 Validation & Business Rules
1. **Financial Data Isolation:** Supervisors and Employees must never see monetary payroll amounts, individual salaries, or factory labor cost numbers on any dashboard widget.
2. **Dashboard Query Performance:** Live dashboard summary metrics must compute in $< 500\text{ ms}$ using database indexed aggregation views.

### 9.5 Permissions
* **Super Admin / Factory Admin:** Full access to executive and operational metrics.
* **Accountant:** Full access to financial, payroll, and advance widgets; read access to attendance summaries.
* **Supervisor:** Restricted exclusively to operational line metrics and subordinate headcount.
* **Employee:** Restricted exclusively to personal records.

### 9.6 Important Edge Cases
* **Biometric Terminal Goes Offline:** Device at Factory Gate 1 loses power or network link. *System Rule:* The dashboard hardware widget instantly updates the terminal badge to `OFFLINE` in high-contrast red and displays the elapsed time since the last heartbeat.

### 9.7 Acceptance Criteria
* `GIVEN` a supervisor logging into the dashboard, `WHEN` the home screen loads, `THEN` the UI displays only line staffing, today’s team attendance, and pending leave requests, with zero financial salary cards visible.

---

## Module 10: Reports and Statutory Exports

### 10.1 Purpose & Main Features
Generates compliant, print-ready registers, statutory government inspection forms, internal accounting summaries, and raw data exports in PDF, Excel (.xlsx), and CSV formats.

* **Main Features:**
  - **Pakistan Statutory Labor Registers:**
    - **Form G (Adult Worker Register):** Mandatory statutory register under Section 41 of the Factories Act 1934 (details worker name, father's name, CNIC, nature of work, assigned shift).
    - **Form H (Overtime Register):** Statutory register of overtime worked and payments made under Section 47.
  - **Operational Registers:** Monthly Consolidated Attendance Timesheet, Daily Absenteeism Report, Late Arrival Summary.
  - **Payroll & Accounting Registers:** Master Monthly Salary Register, Cash Denomination Breakdown Sheet, Bank EFT Transfer Schedules, EOBI Monthly Contribution Schedule, Provincial Social Security (SESSI/PESSI) Schedule.
  - Multi-format export: Formatted PDF for printing and signature, Excel/CSV for accounting integration.

### 10.2 User Actions
* **Factory Admin / Accountant:** Select report type, apply date range and department filters, preview data grid, download PDF/Excel/CSV.
* **Supervisor:** Generate line attendance and daily shift report for their assigned team.

### 10.3 Required Data Fields (Export Filters & Parameters)
* `report_type`: Enum selecting desired register.
* `date_range_start`, `date_range_end`: Filter interval.
* `department_id`: Optional filter for specific production line or division.
* `employment_type`: Filter by Permanent, Contract, Daily Wage, or Piece-Rate.
* `export_format`: `PDF`, `XLSX`, `CSV`.

### 10.4 Validation Rules
1. `date_range_end` cannot precede `date_range_start`.
2. Financial and wage reports cannot be generated or downloaded by Supervisor or Employee roles (HTTP `403 Forbidden`).
3. Reports spanning $> 1,000$ employee-months must process asynchronously to prevent HTTP gateway timeouts.

### 10.5 Permissions
* **Factory Admin & Accountant:** Access to all operational, statutory, and financial reports.
* **Supervisor:** Operational attendance and line coverage reports only.
* **Super Admin:** Access to audit trail and security reports.
* **Employee:** Download personal payslip PDF only.

### 10.6 Important Edge Cases
* **Labor Inspector Audit:** Factory inspector requests Form G and Form H for the previous quarter. *System Rule:* System generates a pre-formatted, printable PDF adhering exactly to the statutory layout specified in the Factories Act rules with official factory header and signature spaces.

### 10.7 Acceptance Criteria
* `GIVEN` a request for the Monthly Salary Register, `WHEN` generated by the Accountant, `THEN` the Excel export includes columns for Basic Pay, Allowances, OT Hours, OT Amount, EOBI, Advance Deductions, Net Pay, and Signature/Thumb-impression columns.

---

## Module 11: Notification Engine

### 11.1 Purpose & Main Features
Delivers automated, event-driven operational alerts across multiple channels (In-app notification feed, Email, and SMS/WhatsApp Gateway), ensuring floor personnel and management remain informed of urgent factory events.

* **Main Features:**
  - Multi-Channel Dispatch: In-app notification bell, Email (for administrative staff), and SMS/WhatsApp API (for floor workers without corporate email).
  - Event Triggers:
    - *Attendance:* Unexcused absence after shift grace period, missing check-out alert, consecutive late arrival warnings.
    - *Leave:* New application submitted, supervisor approval/rejection, critical line staffing deficit.
    - *Payroll:* Advance (*Peshgi*) approved, monthly payslip generated, payroll batch locked.
    - *Hardware:* Biometric terminal connection loss alert.
  - Digesting & Rate-Limiting: Grouping high-frequency floor alerts into 30-minute supervisor digests.

### 11.2 User Actions
* **All Users:** View in-app notifications, mark notifications as read, configure personal alert preferences (where permitted).
* **Factory Admin:** Broadcast factory-wide announcements (e.g., Eid holiday schedule, plant maintenance closures).

### 11.3 Required Data Fields
| Field Name | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `notification_id` | UUID | Mandatory | Primary Key. |
| `recipient_user_id`| UUID | Mandatory | FK to `users.user_id`. |
| `channel` | ENUM | Mandatory | `IN_APP`, `EMAIL`, `SMS`, `WHATSAPP`. |
| `event_type` | VARCHAR(50) | Mandatory | E.g., `LEAVE_STATUS`, `LATE_WARNING`, `PAYSLIP_READY`. |
| `title` | VARCHAR(120) | Mandatory | Notification headline. |
| `body` | TEXT | Mandatory | Complete message text (English or Urdu). |
| `is_read` | BOOLEAN | Mandatory | Default `FALSE`. |
| `delivery_status` | ENUM | Mandatory | `PENDING`, `SENT`, `FAILED`. |
| `sent_at` | TIMESTAMPTZ | Optional | Timestamp of dispatch. |

### 11.4 Validation Rules
1. SMS/WhatsApp alerts cannot be dispatched without a valid 11-digit Pakistani mobile number (`03xx-xxxxxxx`).
2. High-volume events (e.g., 20 stitchers clocking in late) must be aggregated into a single consolidated digest for the line supervisor rather than 20 individual SMS alerts.

### 11.5 Permissions
* **Factory Admin:** Send factory-wide broadcasts; view delivery logs.
* **All Users:** View and manage own in-app notifications.

### 11.6 Important Edge Cases
* **SMS Gateway Offline / Balance Exhausted:** If the third-party SMS provider fails, the system logs the failure in `audit_logs` without interrupting the core attendance or payroll transaction, queuing the notification for automatic retry.

### 11.7 Acceptance Criteria
* `GIVEN` a supervisor approving an employee's leave request, `WHEN` approved, `THEN` the employee immediately receives an in-app notification and an SMS alert confirming `Your leave for [Date] has been approved`.

---

## Module 12: Audit Logs and System Settings

### 12.1 Purpose & Main Features
Provides an immutable, tamper-proof audit trail for regulatory compliance and dispute resolution, alongside centralized administration of factory-wide configuration settings.

* **Main Features:**
  - Append-Only Immutable Audit Trail: Captures every create, update, delete, manual punch override, salary modification, and user login event.
  - Write-Once-Read-Many (WORM) storage architecture preventing modification or deletion of log rows by any user, including Super Admin.
  - Detailed change delta capture (`old_state_json` vs. `new_state_json`).
  - System Settings Engine: Factory working days, holiday calendar, provincial labor jurisdiction parameters, grace periods, and backup triggers.

### 12.2 User Actions
* **Super Admin / Factory Admin:** Search and filter audit logs by actor, entity, date range, or action type; update system configuration parameters; trigger on-demand database backup.
* **All Other Roles:** No access to audit logs or system settings.

### 12.3 Required Data Fields
| Field Name | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `audit_id` | UUID | Mandatory | Primary Key. |
| `actor_user_id` | UUID | Optional | FK to `users.user_id` (NULL if system automated task). |
| `action` | ENUM | Mandatory | `CREATE`, `UPDATE`, `DELETE`, `OVERRIDE`, `APPROVE`, `LOGIN`, `EXPORT`. |
| `entity_type` | VARCHAR(50) | Mandatory | Target table (e.g., `attendance_records`, `payslips`). |
| `entity_id` | UUID | Mandatory | Primary Key of the impacted record. |
| `old_state_json` | JSONB | Optional | Complete snapshot of record prior to modification. |
| `new_state_json` | JSONB | Optional | Complete snapshot of record after modification. |
| `ip_address` | INET | Mandatory | Client IP address. |
| `user_agent` | TEXT | Mandatory | Client browser / device user agent. |
| `timestamp` | TIMESTAMPTZ | Mandatory | Cryptographically synchronized UTC timestamp. |

### 12.4 Validation Rules
1. The `audit_logs` table must reject any SQL `UPDATE` or `DELETE` statement at the database trigger level.
2. System setting changes (e.g., changing overtime multipliers or minimum wage rates) take effect only for subsequent transactions and cannot alter locked historical payroll data.

### 12.5 Permissions
* **Super Admin:** Full view access to audit logs and tenant configurations.
* **Factory Admin:** View access to factory-level audit traces; update operational settings.
* **Supervisor / Accountant / Employee:** Strictly forbidden from accessing audit tables.

### 12.6 Important Edge Cases
* **Dispute on Manual Punch Correction:** Worker claims supervisor falsely marked them absent. *Resolution:* The Factory Admin queries the audit log for that `attendance_id`. The log displays the exact timestamp, supervisor's user ID, IP address, and the old vs. new state, providing indisputable evidence for HR resolution.

### 12.7 Acceptance Criteria
* `GIVEN` any user modifying an employee's salary structure, `WHEN` the transaction commits, `THEN` an audit log record is created containing the previous salary and new salary in `old_state_json` and `new_state_json`.
* `GIVEN` a Super Admin executing `DELETE FROM audit_logs;`, `WHEN` executed, `THEN` the database raises an exception and aborts the command.

---

## Complete Role Permissions Matrix (Master Summary)

| Functional Capability | Super Admin | Factory Admin | Supervisor | Accountant | Employee |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **System Settings & Backups** | **CRUD** | View / Edit Config | None | None | None |
| **Audit Logs Inspection** | **Full View** | **Factory Scope** | None | None | None |
| **User Account Management** | **CRUD** | **CRUD** (Plant) | None | None | Self (Pass) |
| **Department & Designations** | **CRUD** | **CRUD** | View Only | View Only | View Only |
| **Employee Master Profiles** | **CRUD** | **CRUD** | View (Team) | View / Edit Bank | View (Self) |
| **Biometric Punch Ingestion** | Full | Full | Read (Team) | Read Only | View (Self) |
| **Attendance Regularization** | Full | Final Approval | Submit / View | Read Only | View (Self) |
| **Shift Definition & Rosters**| **CRUD** | **CRUD** | Assign (Team) | Read Only | View (Self) |
| **Leave Quotas & Policy** | **CRUD** | **CRUD** | None | None | None |
| **Leave Approval Flow** | Full | Final Override | First-tier Approve | Read Only | Apply / Cancel |
| **Salary Structure & Peshgi** | **CRUD** | Approve Policy | None | **CRUD** | None |
| **Payroll Calculation & Lock**| Full | Final Lock | None | Execute / Draft | None |
| **Payslips & Denomination** | View All | View All | None | Full Generation | **Self Only** |
| **Overtime Authorization** | Full | Final Audit | Verify / Propose | Read Only | View (Self) |
| **Executive Dashboards** | Full | Full | Line Board Only | Finance Board | Self View |
| **Statutory Forms (Form G/H)** | Full | Export | None | Export | None |

---

## Architectural Sign-off & Verification

```
Prepared By: ___________________________ (Lead Business Analyst & Systems Architect)
Reviewed By: ___________________________ (Factory Operations Director)
Approved By: ___________________________ (Head of HR & Plant Finance)
Date:        ___________________________
```
