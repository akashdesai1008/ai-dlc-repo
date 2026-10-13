# Product Requirements Document: Patient Appointment Booking MVP

## 1. Overview

The clinic needs a web application that lets patients book appointments and lets clinic staff manage appointment availability and bookings. The application replaces the clinic's phone-based process to reduce scheduling conflicts.

All practice data used by the MVP must be fictional.

## 2. Goals

- Let patients independently find and book an available doctor appointment.
- Prevent more than one patient from booking the same doctor time slot.
- Give patients visibility and control over their upcoming appointments.
- Give clinic staff visibility and control over clinic availability and bookings.

## 3. Users and access

| User | Access |
| --- | --- |
| Patient | Sign in, view doctors and available slots, book a slot, and view or cancel only their own upcoming appointments. |
| Clinic staff | Sign in, create available slots, view all bookings, filter bookings by date and doctor, and mark appointments completed or cancelled. |

The application must enforce role-based access. A patient must be denied access to another patient's appointment data.

## 4. Scope

### In scope

- Role-based sign-in for assigned patient and clinic-staff accounts.
- Doctor directory with name and specialty.
- Staff-created, 30-minute appointment slots.
- Patient booking, appointment list, and cancellation.
- Staff booking list, date and doctor filters, and status management.

### Out of scope

- Payments
- Insurance verification
- Medical records
- Prescriptions
- Email or SMS notifications

## 5. Functional requirements

### 5.1 Authentication and authorization

1. The system must authenticate patients and clinic staff using assigned accounts.
2. The system must determine available actions from the signed-in user's role.
3. The system must authorize every appointment read or update; client-side hiding alone is insufficient.
4. A patient may retrieve only appointments whose patient identifier matches the signed-in patient.

### 5.2 Doctors and availability

1. Patients must be able to view each doctor's name and specialty.
2. Clinic staff must be able to create an available slot for a doctor.
3. Each slot must last exactly 30 minutes.
4. Patients must be able to view only slots that are available and have not started.

### 5.3 Booking

1. A patient must be able to select and book one available slot.
2. Booking must atomically reserve the selected slot so that only one patient can book it.
3. If a slot has already been reserved when the booking is submitted, the booking must fail and display: **“This slot is no longer available.”**
4. On successful booking, the system must display a confirmation and show the appointment in the patient's appointment list.

### 5.4 Patient appointment management

1. Patients must be able to view their own upcoming appointments and their statuses.
2. Patients must be able to cancel an appointment only before its start time.
3. When a patient cancels an upcoming appointment, the appointment status must remain visible as `cancelled` and its slot must become available for another patient to book.

### 5.5 Clinic staff appointment management

1. Clinic staff must be able to view all bookings.
2. Clinic staff must be able to filter bookings by appointment date and doctor.
3. Clinic staff must be able to mark an appointment as `completed` or `cancelled`.
4. A staff cancellation must leave the appointment visible with a `cancelled` status and make a future slot available again.

## 6. Business rules

| Rule | Requirement |
| --- | --- |
| Duration | Every appointment slot is 30 minutes. |
| Exclusivity | A doctor time slot can have at most one active booking. |
| Past slots | A slot at or before its start time cannot be booked. |
| Patient cancellation | A patient can cancel only before the appointment start time. |
| Reavailability | Cancelling a future appointment makes its slot available again. |
| Privacy | Patients cannot access another patient's appointments. |
| Cancellation record | Cancelled appointments remain visible with a cancelled status. |

An active booking is one whose status is not `cancelled`. A completed appointment is historical and cannot be booked or cancelled by a patient.

## 7. Primary user flows

### Patient books an appointment

1. The patient signs in.
2. The patient browses doctors by name and specialty.
3. The patient selects a future available 30-minute slot.
4. The system verifies that the slot is still available and atomically creates the booking.
5. The system confirms the booking and adds it to the patient's appointment list.

If another patient has booked the slot first, the system does not create a booking and displays **“This slot is no longer available.”**

### Patient cancels an appointment

1. The patient signs in and opens their upcoming appointments.
2. The patient selects an appointment that has not started.
3. The system marks the appointment as `cancelled`.
4. The cancelled appointment remains in the patient's list, and the future slot becomes available.

### Staff manages bookings

1. A staff member signs in.
2. The staff member creates 30-minute available slots for doctors.
3. The staff member views all bookings, optionally filtering by date and doctor.
4. The staff member marks an appointment as `completed` or `cancelled` as appropriate.

## 8. Data requirements

| Entity | Required fields |
| --- | --- |
| User | Identifier, fictional name, role (`patient` or `clinic_staff`) |
| Doctor | Identifier, fictional name, specialty |
| Appointment slot | Identifier, doctor identifier, start time, end time, availability |
| Appointment | Identifier, slot identifier, patient identifier, status (`booked`, `completed`, or `cancelled`), created time |

The system must derive a slot's end time as 30 minutes after its start time. Appointment status changes must preserve the appointment record for display and auditability within the MVP.

## 9. Acceptance criteria

1. A successful booking displays a confirmation and appears in the signed-in patient's appointment list.
2. When two patients try to book the same slot, exactly one booking succeeds; the other patient sees **“This slot is no longer available.”**
3. A cancelled appointment remains visible with a `cancelled` status.
4. Clinic staff can filter appointments by date and doctor.
5. Unauthorized access to another patient's appointment is denied.

## 10. Success measures

- No duplicate active bookings exist for the same doctor slot.
- Patients can complete booking and cancellation without contacting clinic staff.
- Clinic staff can find bookings for a selected date and doctor.
