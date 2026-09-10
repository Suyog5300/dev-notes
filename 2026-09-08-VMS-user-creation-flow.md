Same database — you just scrolled to different columns
No, I was never looking anywhere else — same vms_db.users table both times. My very first query on this table actually already included is_active in the columns I selected (all rows showed 1). Your new screenshot just scrolled right to show columns I hadn't displayed before (address, dob, is_active, etc.) — same 5-6 rows, same table.

Why "Block" and "is_active" don't look connected — because they're not
This is the real source of confusion, and it's worth being precise about it: there are actually three separate active/blocked-style flags in this system, spread across two different databases:

vms_db.users.is_active (what you're looking at in pgAdmin right now) — lives in the company database, alongside the user's name/mobile/email. Right now this only controls whether someone shows up in the Users list at all (getUsers filters where isActive = 1) — nothing currently lets you toggle it from the UI.
iot_erp_master_db.user_logins.is_active — a different column, same name, but in the master database's login-credentials table. This one has real teeth: login checks it directly and throws UserInActive if it's 0, genuinely blocking sign-in.
iot_erp_master_db.user_logins.is_blocked — also in the master database. This is what the current "Block"/"Unblock" buttons touch. It's a separate lockout concept (also auto-set after too many failed password attempts) from "is this account active."
So when you say "is_active, which is like enable/disable" — you're right that it should mean that, but there are two different is_active columns in two different databases, and neither is currently wired to a button. The one that actually blocks login on its own (user_logins.is_active) is the one I'd recommend wiring "Enable/Disable" to, if the goal is "can this person log in at all" — that's a real, meaningful toggle, distinct from the existing temporary-lockout Block/Unblock. Want me to add that as a genuinely new toggle, separate from Block/Unblock (not a replacement — they do different jobs), or do you want to think about it more first?

employees — pre-existing, not something I created, and genuinely a different table for a different purpose
I didn't create it — I found it (and the trigger that touches it) while debugging the earlier crash, already sitting in vms_db. Just confirmed: it's referenced by workforce.service.ts, which is a CMS/workforce-scheduling concern (it has fields like skillset, availability_status, budget — nothing VMS cares about). My best read: vms_db was likely created from a shared template used across the whole Digineous ERP suite, which is why tables belonging to modules VMS doesn't even use (like employees) still exist here — same pattern as the Employees entity shared between VMS and HRMS mentioned earlier in this engagement. users (people who log into VMS) and employees (a workforce roster for CMS-style features) aren't duplicating the same data on purpose — they're two unrelated concepts that happen to both exist in this database.

The full flow, step by step
You click "Add User", fill in the form, and submit.
Backend creates a row in vms_db.users — the profile (name, mobile, email).
Backend creates a row in iot_erp_master_db.user_logins — the actual login credentials (username, hashed password).
Backend creates a row in vms_db.user_roles — linking that user to the role you picked (e.g., role_id 4 for Employee).
The moment step 4's insert happens, Postgres itself — not your application code — automatically runs insert_resource_if_role() because that function is registered as a trigger on user_roles. Nothing in the Node.js backend calls it directly; the database does it on its own as a side effect of the insert.
That trigger checks: is the role_id 3, 4, or 5? If yes, it looks up the new user's name and the role's description, and inserts a matching row into employees — because CMS's workforce features apparently expect certain roles to also exist there.
What was broken in that trigger, in order of discovery:

First bug: step 6's insert had u.user_id::varchar — taking the user's ID (a UUID) and converting it to plain text before inserting it into employees.user_id, which is itself a UUID column. Postgres refuses to put text into a UUID column without an explicit "yes, treat this text as a UUID" instruction, so it crashed with a type mismatch. Fix: removed the conversion — the ID was already the correct type; converting it was the bug, not a missing conversion elsewhere.
Second bug: step 6 also copies the role's description into employees.employee_type, a column capped at 50 characters. Your "Employee" role's description was 56 characters, so it overflowed and crashed. Fix: cut it to the first 50 characters instead of crashing (LEFT(...)), so no future role's description — however long — can cause this same crash again.
Both bugs live inside that one trigger function, inside vms_db only — nothing about them touches your application code or any other tenant's database.
