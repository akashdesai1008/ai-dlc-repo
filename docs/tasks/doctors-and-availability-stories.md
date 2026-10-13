# Doctors and Availability Development Stories

Add each story below to the Doctor Appointment kanban board. Stories are ordered by dependency; testing stories can be completed alongside their corresponding implementation story.

## Story 1: Create doctor and appointment-slot tables

**As a** database developer, **I want** the doctor and appointment-slot tables created **so that** the application can persist the doctor directory and staff-created availability.

### Acceptance criteria

- Create `doctors` with `id`, required `name`, and required `specialty`.
- Create `appointment_slots` with `id`, `doctor_id`, `start_at`, `end_at`, `availability`, and `created_by`.
- Enforce the doctor foreign key, `start_at < end_at`, permitted availability values, and uniqueness of `(doctor_id, start_at)`.
- Store timestamps in UTC.

## Story 2: Create availability database routines

**As a** database developer, **I want** stored procedures for slot creation and future-availability retrieval **so that** database access consistently applies the availability rules.

### Acceptance criteria

- A slot-creation procedure accepts a doctor, start time, and staff creator, derives an end time exactly 30 minutes later, and creates an `available` slot.
- The procedure rejects unknown doctors, starts at or before the database's current UTC time, and duplicate `(doctor_id, start_at)` values.
- A retrieval procedure returns only `available` slots whose start time is in the future, with optional doctor and date filters.

## Story 3: Build the Spring Boot Doctors and Availability API

**As an** authenticated user, **I want** secure doctor and availability endpoints **so that** I can view doctors and future slots, while only clinic staff can create slots.

### Acceptance criteria

- `GET /doctors` returns each doctor's identifier, fictional name, and specialty to authenticated users.
- `GET /slots?doctorId=&date=` returns only future available slots and supports the optional filters.
- `POST /slots` is restricted to clinic staff and uses the database routine to create the slot.
- The API rejects invalid input, unknown doctors, past or current start times, and duplicate slots; patients receive an authorization failure for `POST /slots`.

## Story 4: Build the Angular Doctors and Availability UI

**As a** patient or clinic-staff user, **I want** role-appropriate doctor and availability views **so that** I can find future appointments or create available slots.

### Acceptance criteria

- The doctor directory displays each fictional doctor name and specialty.
- The slot list displays future available slots obtained from the API.
- The clinic-staff form selects a doctor and start time, explains the fixed 30-minute duration, and submits to `POST /slots`.
- The UI shows API validation and authorization failures; route guards may tailor navigation but do not authorize requests.

## Story 5: Test the database schema and availability routines

**As a** quality engineer, **I want** database tests for schema constraints and routines **so that** invalid or unavailable slots cannot be persisted or returned.

### Acceptance criteria

- Tests verify required doctor fields, the doctor foreign key, the slot uniqueness constraint, valid availability values, and `start_at < end_at`.
- Tests verify that slot creation derives a 30-minute end time and rejects unknown doctors, non-future starts, and duplicates.
- Tests verify retrieval excludes unavailable and past slots and applies the optional filters.

## Story 6: Test the Spring Boot API

**As a** quality engineer, **I want** API tests for doctor and availability endpoints **so that** authorization, validation, and responses match the technical requirements.

### Acceptance criteria

- Tests verify authenticated users can retrieve doctors and future available slots.
- Tests verify clinic staff can create a valid slot with a derived 30-minute end time.
- Tests verify patients cannot create slots and invalid, unknown, past/current, or duplicate slot requests fail.
- Tests verify a client-supplied end time cannot change the derived duration.

## Story 7: Test the Angular Doctors and Availability UI

**As a** quality engineer, **I want** Angular UI tests **so that** the application presents the correct views and safely surfaces API failures.

### Acceptance criteria

- Tests verify the doctor directory renders names and specialties and the slot list renders API-provided future slots.
- Tests verify clinic staff can select a doctor and start time, see the 30-minute duration, and submit the form.
- Tests verify patient navigation does not expose slot creation and API validation or authorization failures are displayed.
