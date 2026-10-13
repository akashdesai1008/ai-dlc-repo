# Technical Requirements Document: Doctors and Availability

## 1. Purpose and scope

This document defines the technical design for section 5.2, “Doctors and availability,” of the [Patient Appointment Booking MVP product requirements document](../patient-appointment-booking-prd.md). It covers the doctor directory, staff creation of 30-minute available slots, and patient visibility of future available slots.

Booking, appointment lifecycle management, payments, insurance, medical records, prescriptions, and notifications are out of scope for this document and require separate TRDs.

## 2. Architecture

Implement this capability as an Angular single-page application, Spring Boot REST API, and relational database.

- Angular displays doctor and slot data through dedicated services and renders patient and clinic-staff views appropriate to the authenticated role.
- Spring Boot owns request validation, authorization, slot creation, and availability queries. The Angular client must not be trusted to enforce access rules.
- Spring Security authenticates requests and exposes the authenticated user ID and role to Spring MVC controllers and services.
- The relational database stores doctors and appointment slots.
- Store timestamps as UTC and convert them to the clinic's display timezone only at the Angular client boundary.

## 3. Data model

| Table | Required columns | Constraints |
| --- | --- | --- |
| `doctors` | `id`, `name`, `specialty` | Name and specialty are required; names are fictional. |
| `appointment_slots` | `id`, `doctor_id`, `start_at`, `end_at`, `availability`, `created_by` | `doctor_id` references `doctors`; `created_by` identifies the staff creator; `availability` is `available` or `unavailable`; `end_at = start_at + 30 minutes`; `start_at < end_at`; `(doctor_id, start_at)` is unique. |

The Spring Boot service must derive `end_at` from `start_at`; it must not trust a client-provided end time. A slot is visible to patients only when `availability` is `available` and `start_at` is later than the current server time. A subsequent booking TRD will define how booking changes slot availability.

## 4. Authorization

| Capability | Patient | Clinic staff |
| --- | --- | --- |
| View doctors | Allowed | Allowed |
| View future available slots | Allowed | Allowed |
| Create an available slot | Denied | Allowed |

Spring Security must enforce these permissions in the API. Angular route guards may tailor navigation by role, but do not replace API authorization.

## 5. API requirements

| Method and path | Role | Behavior |
| --- | --- | --- |
| `GET /doctors` | Authenticated user | Returns each doctor's identifier, fictional name, and specialty. |
| `GET /slots?doctorId=&date=` | Authenticated user | Returns only slots with `availability = available` and `start_at` later than the current server time, optionally filtered by doctor and date. |
| `POST /slots` | Clinic staff | Validates the doctor and start time, derives an end time exactly 30 minutes later, and creates an available slot. |

`POST /slots` must reject an attempt to create a slot for an unknown doctor, an invalid start time, a slot at or before the current server time, or an existing `(doctor_id, start_at)` pair. The API must return an authorization failure when a patient attempts to create a slot.

## 6. User-interface requirements

- The Angular doctor directory displays each doctor's fictional name and specialty.
- The patient slot list requests and displays only future available slots.
- The clinic-staff slot creation form selects a doctor and start time; it communicates that every created slot lasts 30 minutes.
- The UI displays API validation and authorization failures without treating client-side visibility rules as authorization.

## 7. Acceptance verification

| PRD requirement | Technical verification |
| --- | --- |
| Patients view a doctor's name and specialty | Request the doctor directory as a patient and assert each result has the configured name and specialty. |
| Staff create an available slot | Submit a valid slot as staff and assert the stored slot is `available`, is associated with the selected doctor, and has the derived end time. |
| Slots last exactly 30 minutes | Create a slot and assert `end_at - start_at` equals 30 minutes; submit a client end time and assert it cannot alter the derived duration. |
| Patients view only future available slots | Seed available, unavailable, and past slots; assert the patient query returns only future available slots. |
