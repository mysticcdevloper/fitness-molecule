# Security Specification: Fitness Molecule Firebase Security TDD

## 1. Data Invariants
- **Identity Integrity**: All writes (`registrations`, `trainerBookings`, `classBookings`, `enquiries`) must have `userId` matching the authenticated user's `uid`. No user can create or update a sub-document with another user's ID.
- **Self-Assigned Avoidance**: Users cannot access or modify profiles belonging to other `uid` paths.
- **Immutable Fields**: `createdAt`, `userId`, and primary plan or trainer configurations must remain unchanged during updates.
- **PII Isolation**: Complete restriction of read access on PII records to the authenticated owner (`uid`).
- **Temporal Validity**: Timestamp metadata must be validated or keep consistent structure.

## 2. The "Dirty Dozen" Payloads
These payloads attempt to exploit access gaps, escalate privileges, swap identities, or inject toxic data strings:

1. **User Identity Spoofing**: An authenticated user `attacker1` tries to write a User Profile under `users/victim_user`.
2. **Ghost User Privilege Escalation**: A user payload attempting to claim admin status in `users/{userId}`.
3. **Orphaned Registration (No User Auth)**: Trying to write a membership plan without any active Firebase Authentication token.
4. **Registration Overwrite (ID Swap)**: An authenticated user `user_A` trying to book a plan under `registrations/reg_123` but setting the `userId` field to `user_B`.
5. **PII Blanket Scraping**: Attempting to execute a query to download all user documents in `users` or membership details in `registrations` of other members.
6. **Toxic Document ID Injection**: Attempting to write a trainer session with a bloated 1KB document ID containing garbage/special characters.
7. **Negative Age Constraint Exploitation**: Creating an enquiry where the client's `age` is `-55` or `1000`.
8. **Empty Required Client Fields**: Subelement booking missing the mandatory `clientEmail` or `fullName` values.
9. **Spaghetti Update of Immutables**: Modifying a verified `createdAt` fields in an active plan registration.
10. **State Shortcutting/Terminal Hijack**: Re-submitting a booking state to change key configurations like `planPrice` or `trainerId` after confirmation.
11. **Excessive String Flooding**: Submitting a booking or notes field containing a 500KB heavy string to overwhelm local resources and deplete the Spark database quota.
12. **Double Booking Modification (Relational Shift)**: Modifying a class slot reservation that changes the primary `classId` but keeps the original ID, creating a split relation.

## 3. Verified Rules Design Plan
We will draft a zero-trust `firestore.rules` containing global validators of types, IDs, and ownership checks.
`firestore.rules` will enforce that only the respective `userId` can read or write their own registrations and scheduled sessions.
