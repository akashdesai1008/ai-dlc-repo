# Technical Requirements Document: Patient Appointment Booking MVP

## 1. Purpose and scope

This document defines the technical design for the Patient Appointment Booking MVP described in the [product requirements document](../patient-appointment-booking-prd.md). The system supports fictional patients, clinic staff, doctors, 30-minute appointment slots, booking, cancellation, and staff booking management. Payments, insurance, medical records, prescriptions, and notifications are excluded.

## 2. Architecture

Implement the application as an Angular single-page application, Spring Boot REST API, and relational database.

- Angular renders separate patient and clinic-staff views from the authenticated user's role and calls the API through Angular services.
- Spring Boot owns authorization, validation, booking transactions, and status changes. The Angular client must not be trusted to enforce access rules.
- Spring Security authenticates requests and exposes the authenticated user ID and role to controllers and services.
- The relational database stores users, doctors, slots, and appointment history and enforces booking exclusivity.
- Store timestamps as UTC and convert them to the clinic's display timezone only at the client boundary.

## 3. Data model

| Table | Required columns | Constraints |
| --- | --- | --- |
| `users` | `id`, `name`, `role` | `role` is `patient` or `clinic_staff`; names are fictional. |
| `doctors` | `id`, `name`, `specialty` | Name and specialty are required. |
| `appointment_slots` | `id`, `doctor_id`, `start_at`, `end_at`, `created_by` | `doctor_id` and `created_by` reference their respective records; `end_at = start_at + 30 minutes`; `start_at < end_at`; `(doctor_id, start_at)` is unique. |
| `appointments` | `id`, `slot_id`, `patient_id`, `status`, `created_at`, `updated_at` | `slot_id` and `patient_id` reference their respective records; `status` is `booked`, `completed`, or `cancelled`. |

Slot availability is derived rather than independently mutable: a slot is available when its start time is in the future and it has no appointment whose status is `booked` or `completed`. Cancelled appointments are retained and do not make a slot unavailable.

The database must prevent duplicate active bookings with a partial unique index:

```sql
CREATE UNIQUE INDEX appointments_one_active_booking_per_slot
  ON appointments (slot_id)
  WHERE status IN ('booked', 'completed');
```

## 4. Authorization

Spring Security establishes a server-side identity containing the user ID and role. Every Spring MVC controller and service must derive the acting user from that identity and never accept an actor or patient ID from the Angular client as authorization evidence.

| Capability | Patient | Clinic staff |
| --- | --- | --- |
| View doctors and future available slots | Allowed | Allowed |
| Create a slot | Denied | Allowed |
| Create a booking | Allowed for self | Denied |
| View appointments | Own appointments only | All appointments |
| Cancel an appointment | Own, before start time only | Any appointment |
| Mark an appointment completed | Denied | Allowed |

Patient appointment queries must include `patient_id = authenticated_user_id`. Direct requests for another patient's appointment must return an authorization failure without exposing its data.

## 5. API requirements

| Method and path | Role | Behavior |
| --- | --- | --- |
| `GET /doctors` | Authenticated user | Returns doctor ID, fictional name, and specialty. |
| `GET /slots?doctorId=&date=` | Authenticated user | Returns only future available slots, optionally filtered by doctor and date. |
| `POST /slots` | Clinic staff | Validates the doctor and exact 30-minute duration, then creates a future available slot. |
| `POST /appointments` | Patient | Atomically books one available future slot for the authenticated patient. |
| `GET /appointments` | Patient or staff | Patients receive only their appointments; staff may filter by appointment date and doctor. |
| `PATCH /appointments/{id}/cancel` | Patient or staff | Sets a permitted appointment to `cancelled` without deleting it. |
| `PATCH /appointments/{id}/status` | Clinic staff | Sets a booking to `completed` or `cancelled`. |

The Spring Boot booking service handling `POST /appointments` must run in a `@Transactional` database transaction. It must verify that the slot is future and has no active booking, insert the `booked` appointment, and commit. If a concurrent request inserts first, translate the database unique-constraint violation into a conflict response that the Angular client displays as **“This slot is no longer available.”** The transaction must roll back on every failure.

Cancellation must update an existing appointment rather than delete it. A patient cancellation must reject appointments at or after their start time. Staff cancellation must only make a future slot available again; a past slot remains excluded from availability because availability also requires `start_at > current time`.

## 6. Validation and error handling

- Reject malformed IDs, unknown doctors or slots, invalid status transitions, and slots whose end time is not exactly 30 minutes after start time.
- Reject creating or booking a slot at or before the current server time.
- Use server time for all time comparisons.
- Return an authorization failure for disallowed roles or cross-patient access.
- Return a not-found response only when the requester is permitted to know the resource exists; otherwise return an authorization failure.
- Return a conflict response for a concurrent booking loss and show **“This slot is no longer available.”**

## 7. User-interface requirements

- The doctor directory displays each doctor's name and specialty.
- The patient slot list contains only future available slots.
- A successful booking displays a confirmation and the appointment appears in the patient's appointment list.
- Angular route guards may tailor navigation by role, but Spring Security remains the authorization boundary.
- Patient and staff appointment lists display status, including retained `cancelled` records.
- Staff booking management provides date and doctor filters and controls for `completed` and `cancelled` statuses.

## 8. Acceptance verification

| PRD acceptance criterion | Technical verification |
| --- | --- |
| Successful booking confirms and appears in the patient's list | Book a future slot and assert the response and patient-scoped appointment query contain the booking. |
| Concurrent booking permits exactly one patient | Submit two booking requests for one slot concurrently; assert one success and one conflict with the required message. |
| Cancelled appointment remains visible | Cancel a future booking; assert its status is `cancelled`, it remains in the patient's list, and its slot is returned as available. |
| Staff can filter by date and doctor | Create bookings across doctors and dates; assert staff queries return only matching records. |
| Another patient's appointment is denied | Attempt to read and cancel one patient's appointment using another patient's identity; assert authorization failure and no mutation. |
