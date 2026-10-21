# RF Forms Firebase security setup

RF Forms requires Firebase Email/Password Authentication for all users. Each administrator or staff member must have an assigned account and the permissions configured in Realtime Database. Password-free/anonymous access is not supported.

## Bootstrap the first administrator

Do these steps in the Firebase project that owns the RF Forms Realtime Database. The current database rules deny all client access, so do not publish the new rules until the administrator profile is prepared.

1. In **Authentication → Sign-in method**, enable **Email/Password**. In **Authentication → Settings → Authorized domains**, make sure `rotifactorytesting.github.io` is listed. Do not enable public email/password account creation in the app; create accounts from Firebase Console.
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

## Operational forms and reports

Assigned staff sign in with their work email and password, then use only the forms and reports granted by their profile. The **Template Reports** screen filters saved entries by their chosen form date and supports daily, Monday–Sunday weekly, and calendar-month ranges. Use **Open print-ready sheet / Save as PDF**, then select **Save as PDF** in the browser print dialog. A missing Firebase web configuration prevents the app from connecting; configure the ignored local `firebase-config.js` from `firebase-config.example.js` for local development and never commit real environment-specific configuration.

### WhatsApp submission notifications

To route staff form submissions to an administrator:

1. In **Admin → Staff**, edit the receiving administrator's profile and enter a WhatsApp number with its country code (digits only, for example `919810000000`).
2. Enable **WhatsApp Reports** for that profile and save it. The app prefers an administrator profile with this flag; otherwise it uses the first configured WhatsApp Reports recipient. The same setting receives the daily report.
3. After a staff member saves a form online, RF Forms opens a prefilled WhatsApp chat containing the form name, staff member, submission time, and entered form fields. The staff member must review it and tap **Send** in WhatsApp.

WhatsApp click-to-chat does not permit a browser app to send messages silently or guarantee delivery. If a browser blocks the chat tab, the saved submission remains saved and RF Forms shows an **Open WhatsApp message** action. If no recipient is configured, ask an administrator to add the number and enable **WhatsApp Reports**. The notification contact number is readable by signed-in app users so the existing client-side WhatsApp flows can use it; do not enter a number that should be kept secret.

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
