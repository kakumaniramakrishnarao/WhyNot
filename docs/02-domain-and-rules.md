# 02 — Domain model & rule catalog

## Core concepts

| Concept | Meaning |
| --- | --- |
| **ClinicSnapshot** | Everything the engine needs, frozen: clinic settings, resources, services, policies (one version), patients involved, existing bookings in the window |
| **BookingRequest** | `{ patientId, serviceId, start (instant), practitionerId? }` — what the patient asked for |
| **Candidate** | One concrete way to satisfy a request: `(practitioner, room, equipmentUnit | null)` at the requested time |
| **Rule** | A catalog entry: a kind of constraint (e.g. "practitioner must be within working hours"), with id, version, family, weight, visibility, relaxability |
| **Fact** | A *grounded* rule instance that is violated: rule + subject + parameters. Example: `buffer-after:prac=dr-rao:booking=bk-0142` |
| **Violation matrix** | For each candidate, the set of facts it violates. A request is bookable iff some candidate violates nothing |
| **Explanation** | The minimum-weight set of facts that blocks every candidate (see doc 03) |
| **Alternative** | A modified request that is bookable, with its cost and what changed |
| **Admin fix** | The smallest set of admin-relaxable facts whose relaxation would make the *original* request bookable |

## Entities (snapshot shape)

```ts
Clinic        { id, name, timezone, slotGridMinutes /*15*/, openingHours: WeeklyHours, closedDates: Date[] }
Practitioner  { id, displayName, role, skills: SkillId[], workingHours: WeeklyHours, breaks: WeeklyHours,
                timeOff: { id, from, to, reason /*staff-only*/ }[], dailyCap?: number }
Room          { id, displayName, capabilities: CapabilityId[], maintenance: Window[] }
Equipment     { id, displayName, type: EquipmentTypeId, maintenance: Window[] }
Service       { id, displayName, durationMin, requiredSkill: SkillId, requiredRoomCapability?: CapabilityId,
                requiredEquipmentType?: EquipmentTypeId, bufferAfterMin, roomTurnoverMin,
                eligibility: 'any' | 'returning_only' | 'referral_required' }
Patient       { id, displayName, isReturning: boolean, referrals: ServiceId[] }
Booking       { id, patientId, serviceId, practitionerId, roomId, equipmentId?, start, end, status: 'confirmed' }
Policy        { versionId, minNoticeMin, maxAdvanceDays, patientMaxBookingsPerDay, defaultDailyCap? }
```

`WeeklyHours` is a list of `{ weekday, from: 'HH:mm', to: 'HH:mm' }` in clinic local time. `Window` is `{ id, from, to }` in instants.

## Rule catalog (v1)

Each rule produces facts for specific subjects. **Kind** drives grouping and wording. **Weight** drives which explanation is preferred when several are possible (lower weight = preferred to *mention*; see doc 03). **Visibility** controls what patients see. **Relaxable by** controls whether a fact may appear in alternatives (patient changes their request) or admin fixes (clinic changes policy).

| Rule ID | Family | Grounded per | Violated when | Weight | Visibility | Relaxable by |
| --- | --- | --- | --- | --- | --- | --- |
| `clinic-closed-day` | availability | clinic | Date is a closed date or weekday has no opening hours | 1 | public | patient (time) |
| `clinic-hours` | availability | clinic | Appointment not fully inside opening hours | 2 | public | patient (time) |
| `min-notice` | timing policy | clinic | `start - now < minNoticeMin` | 2 | public | patient (time), admin |
| `max-advance` | timing policy | clinic | `start - now > maxAdvanceDays` | 2 | public | patient (time), admin |
| `patient-eligibility-returning` | patient | patient | Service is returning-only and patient is new | 1 | public (to that patient) | admin |
| `patient-eligibility-referral` | patient | patient | Service needs referral and patient has none | 1 | public (to that patient) | — (needs referral) |
| `patient-daily-limit` | patient | patient | Patient already has `patientMaxBookingsPerDay` bookings that day | 2 | public (to that patient) | patient (date), admin |
| `request-practitioner` | preference | request (one fact) | Patient asked for a specific practitioner and candidate uses another | 4 | public | patient |
| `practitioner-skill` | capability | practitioner | Practitioner lacks the service's required skill | 5 | public ("not offered by") | — |
| `practitioner-hours` | availability | practitioner | Appointment not inside practitioner's working hours that day | 3 | public | patient (time/practitioner) |
| `practitioner-break` | availability | practitioner + break | Overlaps a scheduled break | 3 | public ("unavailable") | patient |
| `practitioner-time-off` | availability | practitioner + time-off | Overlaps time off | 3 | **staff** (patient sees "unavailable") | patient |
| `practitioner-busy` | occupancy | practitioner + booking | Overlaps another booking | 3 | public ("already booked"; never who) | patient |
| `practitioner-buffer` | occupancy | practitioner + booking | Within the *buffer after* an adjacent booking (or this booking's buffer would overlap the next one) but no direct overlap | 3 | public | patient, admin |
| `practitioner-daily-cap` | policy | practitioner + date | Practitioner already at daily cap | 3 | public ("fully booked that day") | patient, admin |
| `room-capability` | capability | room | Room lacks required capability | 6 | staff (patient never sees room-level detail unless *all* rooms fail) | — |
| `room-maintenance` | availability | room + window | Overlaps maintenance | 4 | public ("room unavailable") | patient |
| `room-busy` | occupancy | room + booking | Overlaps another booking in the room | 4 | public | patient |
| `room-turnover` | occupancy | room + booking | Within room turnover time after an adjacent booking | 4 | public | patient, admin |
| `equipment-type` | capability | equipment unit | Unit is not of the required type | 6 | staff | — |
| `equipment-maintenance` | availability | equipment unit + window | Unit out of service | 4 | public ("equipment unavailable") | patient |
| `equipment-busy` | occupancy | equipment unit + booking | Unit in use by another booking | 4 | public | patient |

Notes:

- **Buffer vs. busy are separate rules on purpose.** "Booked" and "within the 15-minute cleaning buffer" are different stories, and only the buffer is a policy the admin could change. This separation powers the hero scenarios.
- **Grounding matters.** `practitioner-busy` is grounded per *booking*, so the explanation can say "already booked 14:00–14:45" without revealing the patient.
- **Capability rules have high weight** so the engine prefers explanations like "Dr. Rao is booked" over "Nurse Iyer can't do this procedure" when both would suffice — but capability facts still appear when they are *needed* (e.g., when the only other candidates are unqualified, "only Dr. Rao and Dr. Mehta offer this" is part of the story).
- Request-validation errors (unknown service, start not on the slot grid, duration overflow) are **not** rules — they return `invalid_request` before the engine runs.
- Adding a rule = add a catalog entry + check function + templates (both audiences) + tests + a golden scenario. Bump the rule's version when its semantics change.

## Riverside Family Clinic (demo data)

Timezone `Europe/London` (exercises DST), 15-minute grid, open Mon–Fri 08:00–18:00, Sat 09:00–13:00.

| Resource | Details |
| --- | --- |
| Dr. Asha Rao (GP) | skills: consult, minor-procedure; Mon–Fri 08:00–16:00; lunch 12:30–13:15 |
| Dr. Vikram Mehta (GP) | skills: consult, minor-procedure; Tue–Sat; on leave for 3 days in demo week |
| Dr. Lena Okafor (GP) | skills: consult; Mon–Thu 10:00–18:00 |
| Nurse Priya Iyer | skills: vaccination, dressing, bloods; Mon–Fri 08:00–14:00 |
| Physio Tom Harris | skills: physio; Mon, Wed, Fri 09:00–17:00; daily cap 6 |
| Sonographer Mei Chen | skills: ultrasound; Tue, Thu 09:00–15:00 |
| Rooms | Consult 1, Consult 2 (consult); Consult 3 (consult, dressing); Treatment Room (procedure, dressing); Physio Studio (physio); Imaging Room (ultrasound) — Treatment Room has a maintenance window in demo week |
| Equipment | Ultrasound unit ×1 (maintenance Thu morning); Cryotherapy unit ×1 |

Services: GP consultation (15 min), Long consultation (30), Minor procedure (30, Treatment Room, cryo unit, buffer 15, turnover 10), Vaccination (10), Wound dressing (20, any dressing-equipped room), Blood test (10), Physio assessment (45, any patient), Physio follow-up (30, returning patients only), Ultrasound scan (30, referral required, Imaging Room, ultrasound unit, turnover 15).

Policy v1: min notice 2 h, max advance 30 days, patient max 1 booking/day, physio daily cap 6.

Patients: ~12 fictional patients, mix of new/returning, some with ultrasound referrals.

The fixture generator (`packages/fixtures/src/clinic.ts`) builds this deterministically relative to a **demo date** (Monday of the demo week) and fills a realistic booking load (~70% occupancy) from a seeded PRNG, then overlays the hand-placed bookings the hero scenarios depend on.

## Hero scenarios (the demo stories)

Unless a scenario says otherwise, `now` = Monday of the demo week, 09:00 clinic time. These are written **before** the engine exists. Golden tests assert them. Each must make a viewer think "a normal booking system couldn't tell me that".

| # | Request | Expected explanation | Expected top alternative |
| --- | --- | --- | --- |
| H1 | Minor procedure, Tue 14:30, any practitioner | Dr. Rao: within 15-min buffer after her 14:00–14:30 procedure. Dr. Mehta: on leave (patient sees "unavailable"). Context line: only Dr. Rao and Dr. Mehta offer minor procedures (capability facts grouped, see doc 04) | Tue 14:45 with Dr. Rao |
| H2 | Ultrasound, Thu 10:00 | Ultrasound unit under maintenance until 12:00 | Thu 12:00 same sonographer |
| H3 | GP consultation, today in 1 hour | Requires 2 hours' notice | Today, earliest slot ≥ 2 h from now |
| H4 | Physio follow-up, new patient | Follow-ups are for returning patients (single, simple reason — proves we don't over-explain) | None — no time change can help, so the panel says so and suggests booking a Physio assessment first (static UI copy for this rule, not an engine alternative). Staff admin fix: the returning-only eligibility rule |
| H5 | Physio Wed 16:00 | Tom Harris has reached his daily limit of 6 | Fri same time; admin fix: raise cap |
| H6 | Wound dressing Tue 11:00 (Nurse Iyer is free) | Both rooms equipped for dressings are unavailable: Treatment Room under maintenance *and* Consult 3 in use — two different reasons that only block together | Tue 11:30 (Consult 3 free again) |
| H7 | GP consultation with Dr. Okafor Wed 09:00 (patient asked for her) | Dr. Okafor starts at 10:00; patient asked for her specifically | Wed 10:00 Dr. Okafor (cost 4) first; Wed 09:00 with Dr. Rao (cost 8) as the other-practitioner alternative — shows the preference trade-off |
| H8 | Blood test, patient already booked that day | You already have an appointment that day (limit 1) | Next day same time |
