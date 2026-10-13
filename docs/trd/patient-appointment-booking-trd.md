# Technical Requirements Document: Patient Appointment Booking MVP

## 1. Purpose and scope

This document defines the technical design for the Patient Appointment Booking MVP described in [the PRD](../patient-appointment-booking-prd.md). The system is a web application with an Angular user interface and a Spring Boot REST API.

The MVP supports assigned patient and clinic-staff accounts, doctor discovery, staff-managed 30-minute availability, appointment booking and cancellation, and staff booking management. It uses fictional practice data only. Payments, insurance, medical records, prescriptions, and notifications are out of scope.

## 2. Architecture

| Layer | Technology | Responsibilities |
| --- | --- | --- |
| Web client | Angular | Sign-in, role-aware navigation, forms, validation feedback, and API integration. |
| API | Spring Boot | Authentication, authorization, business rules, transactional booking, and REST endpoints. |
| Database | Relational database | Durable storage for users, doctors, slots, and appointments. |

The Angular application communicates only with the Spring Boot API over HTTPS. Authorization decisions and booking rules are enforced by the API; the client must not be treated as a security boundary.

## 3. Roles and authorization

| Capability | Patient | Clinic staff |
| --- | --- | --- |
| View doctors and future available slots | Yes | Optional administrative use |
| Book an available slot | Yes, for self only | No |
| View appointments | Own appointments only | All appointments |
| Cancel appointment | Own appointment before it starts | Any appointment |
| Create slots | No | Yes |
| Mark appointment completed or cancelled | No | Yes |

Spring Security must authenticate every protected endpoint and derive the current user and role from the authenticated principal. The API must ignore patient identifiers supplied by a patient client when reading, creating, or changing that patient's appointments.

## 4. Data model

All timestamps must be stored as UTC instants and returned in ISO 8601 format. The UI converts values to the user's local time for display.

### 4.1 Entities

| Entity | Fields |
| --- | --- |
| `users` | `id`, `name`, `role` (`PATIENT`, `CLINIC_STAFF`), authentication identity |
| `doctors` | `id`, `name`, `specialty` |
| `appointment_slots` | `id`, `doctor_id`, `start_time`, `end_time` |
| `appointments` | `id`, `slot_id`, `patient_id`, `status` (`BOOKED`, `COMPLETED`, `CANCELLED`), `created_at` |

`appointment_slots.end_time` must equal `start_time + 30 minutes`. Slot availability is derived rather than stored: a slot is available only when it starts in the future and has no appointment with status `BOOKED` or `COMPLETED`. A cancelled appointment remains stored and does not occupy its future slot.

### 4.2 Constraints and indexes

- Enforce `end_time = start_time + 30 minutes`.
- Enforce one slot per doctor and start time with a unique constraint on `(doctor_id, start_time)`.
- Enforce at most one active appointment per slot with a database partial unique index on `appointments(slot_id)` where `status IN ('BOOKED', 'COMPLETED')`. If the selected database does not support partial indexes, use a transactionally maintained equivalent that prevents two active bookings.
- Index `appointment_slots(doctor_id, start_time)` for availability queries.
- Index `appointments(patient_id, status)` and `appointments(slot_id)` for appointment queries and booking checks.

## 5. API design

All endpoints exchange JSON. Invalid input returns `400 Bad Request`; unauthenticated requests return `401 Unauthorized`; authenticated users without permission receive `403 Forbidden`; missing resources return `404 Not Found`.

### 5.1 Authentication

| Method and path | Access | Purpose |
| --- | --- | --- |
| `POST /api/auth/login` | Public | Authenticate an assigned account and issue the chosen session or token credential. |
| `GET /api/auth/me` | Authenticated | Return the signed-in user's identifier, name, and role. |

The deployed credential transport must use HTTPS. If cookie authentication is selected, cookies must be `HttpOnly`, `Secure`, and use an appropriate `SameSite` setting; state-changing requests also require CSRF protection. If bearer tokens are selected, the API must validate signature, expiry, and role claims.

### 5.2 Doctors and slots

| Method and path | Access | Purpose |
| --- | --- | --- |
| `GET /api/doctors` | Authenticated | List doctor identifiers, names, and specialties. |
| `GET /api/slots?doctorId={id}` | Authenticated | List future available slots, optionally for one doctor. |
| `POST /api/slots` | Clinic staff | Create a future 30-minute slot for a doctor. |

Slot creation accepts `doctorId` and `startTime`; the server derives `endTime`. It rejects unknown doctors, past or current start times, invalid duration input, and a conflicting doctor/start-time pair.

### 5.3 Patient appointments

| Method and path | Access | Purpose |
| --- | --- | --- |
| `GET /api/appointments/me` | Patient | List the current patient's appointments and statuses. |
| `POST /api/appointments` | Patient | Book an available slot using `{ "slotId": "..." }`. |
| `POST /api/appointments/{id}/cancel` | Patient | Cancel the current patient's appointment before its start time. |

`POST /api/appointments` returns `201 Created` with the appointment on success. If the slot is already reserved or has started, it returns `409 Conflict` with an error code such as `SLOT_UNAVAILABLE`; Angular must display **“This slot is no longer available.”**

### 5.4 Staff appointments

| Method and path | Access | Purpose |
| --- | --- | --- |
| `GET /api/staff/appointments?date={YYYY-MM-DD}&doctorId={id}` | Clinic staff | List bookings, optionally filtered by date and doctor. |
| `POST /api/staff/appointments/{id}/status` | Clinic staff | Set status to `COMPLETED` or `CANCELLED`. |

Staff status changes accept `{ "status": "COMPLETED" | "CANCELLED" }`. The API rejects unsupported transitions and must not allow a completed appointment to be reopened or rebooked. Cancelling a future appointment makes its slot available again.

## 6. Booking consistency and lifecycle

The booking service must execute the following in one database transaction:

1. Load the requested slot with a lock appropriate to the database.
2. Verify that the slot exists and its start time is in the future.
3. Verify that no active appointment exists for the slot.
4. Create the `BOOKED` appointment for the authenticated patient.
5. Commit the transaction.

The database active-booking constraint is the final concurrency safeguard. The API must translate a constraint violation or a concurrent reservation into `409 SLOT_UNAVAILABLE`, without exposing database details.

| Current status | Actor | Allowed result |
| --- | --- | --- |
| `BOOKED` | Patient before start time | `CANCELLED` |
| `BOOKED` | Staff | `COMPLETED` or `CANCELLED` |
| `COMPLETED` | Any | No further status change |
| `CANCELLED` | Any | No further status change |

Cancelled appointments remain visible in patient and staff lists. Patient cancellation authorization requires both ownership and a future slot start time.

## 7. Angular application

The Angular application must provide:

- A sign-in route and a session bootstrap call to `/api/auth/me`.
- Role guards for patient and staff routes, with unauthorized users redirected to an access-denied view.
- A doctor directory and availability view that refreshes data after a booking conflict.
- A patient appointment list showing `BOOKED`, `COMPLETED`, and `CANCELLED` statuses and a cancellation control only for future booked appointments.
- A staff view for slot creation, appointment listing, date and doctor filters, and status changes.
- User-facing API error handling, including the required slot-conflict message.

Angular form validation improves usability but does not replace server-side validation. Route guards and hidden controls improve navigation but do not replace API authorization.

## 8. Validation and error handling

The API validates required fields, identifier format, supported status values, UTC timestamp format, future slot start time, and the 30-minute duration rule. It returns a consistent error payload:

```json
{
  "code": "SLOT_UNAVAILABLE",
  "message": "This slot is no longer available."
}
```

Spring Boot must log operational failures with request correlation information where available, while excluding credentials, tokens, and unnecessary patient data from logs and error responses.

## 9. Testing and acceptance mapping

| Requirement | Validation |
| --- | --- |
| Only one patient can reserve a slot | Spring Boot integration test with concurrent booking requests; assert one `201` and one `409`. |
| Patients cannot see or change another patient's appointment | API authorization tests for appointment list, cancellation, and direct appointment identifiers. |
| Patients cannot book or cancel after start time | Service/API tests with past and current slot times. |
| Cancelled appointments remain visible and release future slots | Integration tests for patient and staff cancellation, list response, and subsequent booking. |
| Staff filtering works | API integration tests for date and doctor query parameters; Angular component tests for filter requests. |
| Required booking conflict message is shown | Angular component test for `SLOT_UNAVAILABLE`. |

## 10. Deployment and operations

Configuration must be supplied through environment-specific configuration, not committed credentials. Production must use HTTPS and a managed relational database with backups. Database migrations must create the schema, indexes, and booking constraints before the API serves traffic.

Health endpoints must not disclose sensitive configuration or patient data. Monitoring should record authentication failures, booking conflicts, API error rates, and transaction failures to support safe operation without retaining unnecessary patient information.
