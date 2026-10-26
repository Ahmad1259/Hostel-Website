# Role and Permission Matrix

Status: **Phase 1 draft — awaiting owner approval.**

This is the authoritative list of what each role may do. Backend API checks and
UI visibility must both follow this table. **Every sensitive action must be
checked on the server** — hiding a button in the UI is never a security control.

## Roles

| Role | Scope | Notes |
|---|---|---|
| Platform Admin | Whole platform | Creates/activates hostels, assigns owners. Does not silently alter financial history. |
| Hostel Owner | One or more hostels | Hires managers and wardens for their hostels. |
| Hostel Manager | Assigned hostel(s) | Day-to-day operations, fees, staff reports. |
| Warden | Assigned hostel(s) | Admissions verification, bed assignment, check-in/out. |
| Student | Own records only | Applies, uploads own documents, views own challans/receipts. |

A user may hold one role per hostel plus a platform-level role. Staff access to a
hostel is granted only through an explicit `hostel_staff_assignments` record —
never inferred.

## Permission matrix

| Capability | Admin | Owner | Manager | Warden | Student |
|---|---|---|---|---|---|
| Create/suspend a hostel | Y | N | N | N | N |
| Assign owner to hostel | Y | N | N | N | N |
| Add/remove manager & warden for own hostel(s) | N | Y | N | N | N |
| Edit hostel profile, facilities, policies | N | Y | Y | N | N |
| Publish/edit rooms, beds, rent, deposit | N | Y | Y | Y | N |
| View all student applications for own hostel(s) | N | Y | Y | Y | N |
| Verify/reject application, request documents | N | N | N | Y | N |
| Assign a bed (approved application only) | N | N | N | Y | N |
| Record check-in / check-out | N | N | Y | Y | N |
| Approve room transfer | N | N | Y | Y | N |
| Request room transfer | N | N | N | N | Y |
| Generate monthly challans | N | Y | Y | Y | N |
| Record physical payment | N | N | Y | Y | N |
| Upload payment proof | N | N | N | N | Y |
| Verify/reject payment proof | N | N | Y | Y | N |
| Issue receipt (only after verified payment) | N | N | Y | Y | N |
| View own challans, receipts, assignments | own | own | own | own | Y |
| View audit log | Y | own hostel(s) | own hostel(s) | own hostel(s) | N |
| Export resident personal data | N | N | N | N | N (Admin only, logged) |

## Absolute rules (any role)

1. No role may edit or delete a receipt or a verified payment after issuance.
   Corrections are made by new adjustment records (see data model).
2. No role may assign a bed that is occupied, under maintenance, or not ready.
3. A student sees only their own applications, assignments, challans and
   receipts. Staff see only hostels they are assigned to.
4. Document files are served through signed, expiring, authenticated URLs only;
   never via public paths.
5. Denial by absence: if an action is not listed as allowed, it is denied.
