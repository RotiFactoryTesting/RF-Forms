# RF Forms Firebase security setup

RF Forms supports named, password-free operator access through Firebase Anonymous Authentication, plus Firebase Email/Password Authentication for administrators and staff who need assigned permissions. Anonymous Authentication does not make the database public: operator sessions can create submissions and read only submissions owned by their Firebase UID. Their access persists on the same browser profile while its Firebase session remains available; another browser/device has a different anonymous UID.

## Bootstrap the first administrator

Do these steps in the Firebase project that owns the RF Forms Realtime Database. The current database rules deny all client access, so do not publish the new rules until the administrator profile is prepared.

1. In **Authentication → Sign-in method**, enable **Anonymous** and **Email/Password**. In **Authentication → Settings → Authorized domains**, make sure `rotifactorytesting.github.io` is listed. Do not enable public email/password account creation in the app; create privileged accounts from Firebase Console.
2. In **Authentication → Users**, add the administrator's real email address and a strong, unique password. Copy the new user's **UID**.
3. In **Realtime Database → Data**, edit your existing administrator record under `staff` (the old default record is usually `s0`) and set `authUid` to the new UID. If `staff` has no record yet, add an administrator record such as the following under a new child key like `admin`:

   ```json
   {
     "name": "Administrator",
     "authUid": "REPLACE_WITH_ADMIN_UID",
     "role": "Admin",
     "forms": "ALL",
     "alerts": true,
     "reports": true,
     "canReport": true,
     "canAttendance": true,
     "isAdmin": true,
     "lang": "en",
     "shift": null
   }
   ```

   Do not put the password in the database. Keep any existing staff fields you still need.
4. Add an `access` child keyed by that UID. Example (replace the example UID and staff key with the actual values):

   ```json
   {
     "access": {
       "REPLACE_WITH_ADMIN_UID": {
         "staffKey": "s0",
         "name": "Administrator",
         "role": "Admin",
         "forms": "ALL",
         "alerts": true,
         "reports": true,
         "canReport": true,
         "canAttendance": true,
         "isAdmin": true,
         "lang": "en",
         "shift": null
       }
     }
   }
   ```

   Merge this into the existing data; do not replace the database root or delete existing attendance and submissions.
5. In **Realtime Database → Rules**, publish the contents of [`database.rules.json`](./database.rules.json). All unmatched paths stay denied.
6. Open the hosted app and sign in with the administrator's email and password. The app will remove legacy `pin` fields from staff records after the first admin sign-in.

## Password-free operator use

After Anonymous Authentication is enabled and the rules above are published, an operator can choose **Continue without password**, enter a display name, and submit operational forms. The name is for record attribution only; it is not a password or an identity check. Anonymous operators may read only records created by their current Firebase UID, cannot edit/delete submitted records, and cannot access administrator, attendance, alert-recipient, or report-management features. Keep the administrator email/password sign-in for privileged tasks.

The **Template Reports** screen filters saved entries by their chosen form date and supports daily, Monday–Sunday weekly, and calendar-month ranges. Use **Open print-ready sheet / Save as PDF**, then select **Save as PDF** in the browser print dialog. A missing Firebase web configuration prevents the app from connecting; configure the ignored local `firebase-config.js` from `firebase-config.example.js` for local development and never commit real environment-specific configuration.

## Add staff accounts

1. In **Authentication → Users**, add a staff member using an email address they can access and a strong temporary password. Share it privately and have them change it using **Forgot password?** in the app.
2. In the app, open **Admin → Staff → Add New Staff Member**, enter the Authentication UID and staff details, select only the permissions they need, then save. This creates the staff-to-account link and the protected access profile.
3. Staff sign in once on each device. Subsequent visits on that browser/device use Firebase's saved session. Signing out, clearing browser data, changing devices, or a revoked/expired session requires signing in again.

The sign-in ID is the account email (Firebase Email/Password requires an email-formatted ID). For account recovery, the email must be deliverable. Admins can reset passwords from Firebase Console; they cannot view existing passwords.

## Revoke access and verify

- To remove access immediately, delete `access/{uid}` in Realtime Database. Also disable or delete that account in **Authentication → Users** to prevent future sign-ins.
- To reset a password, use Firebase Console or the app's **Forgot password?** link.
- Keep at least one active administrator. The rules prevent an administrator from removing their own admin access through a client write.
- Staff can submit only forms assigned in their profile; `canReport` grants access to all submissions, and `canAttendance` grants attendance access. Only admins can change profiles, forms, custom fields, and notification contacts.
- In the Rules playground, verify that unauthenticated reads/writes fail; an assigned staff account can read its own profile and submit its assigned forms; an unassigned account cannot read the database; and attendance/report access follows the assigned flags.
- Confirm the deployed app's Firebase `databaseURL` points to the same project/database whose rules you published.

Existing data is not deleted by these rules. A user with `canReport` or admin access can read existing submissions; old submissions are not assigned to an individual Firebase UID, so regular staff only see their own new submissions in history. The app publishes only the WhatsApp numbers needed for configured alerts/reports to signed-in users so those existing notification flows continue to work.
