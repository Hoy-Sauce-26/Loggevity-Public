# Loggevity Privacy Policy

**Effective date:** Sep 18 2026
**Contact:** mattdhoy@gmail.com

---

## The short version

Loggevity does not collect anything. There is no account, no server, and no
analytics. Everything you log stays in an encrypted database on your own
device, and the only way any of it leaves is if you choose to export it.

---

## What Loggevity stores

Everything you enter into the app:

- Activity you log, meaning minutes of exercise, hours of sleep, hours of
  company and time outdoors, along with the dates and times you recorded them
- A short text note attached to an entry, where one is present. The app has no
  field for typing one, so a note only exists if it came in through Import,
  but the database can hold it and an export will carry it back out
- Your answers to the weekly social check-in, if you use that mode
- Your settings: which day your week starts, how socializing is scored and what
  your target is, any category weights you have changed, whether the dashboard
  ring shows pace or raw progress, whether the daily reminder is on and what
  time it fires, and whether you have been shown the first-run tour
- Scores for completed weeks, calculated from the above

That is the complete list.

## Where it is stored

In a single database file inside the app's private storage on your device.

That file is encrypted with SQLCipher. The encryption key is generated on your
device the first time you open the app and is kept in the platform keychain,
which on Android means storage protected by a key held in the Android Keystore.
The key never leaves your device and is not included in any backup.

**Loggevity is excluded from Android's automatic backup and from
device-to-device transfer.** This is deliberate. Because the encryption key
cannot leave your device, a copy of the database restored onto a different
phone could not be read, so rather than hand you a file nothing can open, the
app does not back it up at all. If you want your history to survive a new
phone, use **Your data → Export** and keep the file somewhere you control.

## What Loggevity sends

Nothing.

The app does not hold the `INTERNET` permission. Without it the operating
system will not allow a network connection even if something tried to open
one, and there is no server for the app to talk to in any case.

There are no analytics, no advertising, no crash reporting, no tracking
identifiers, and no third-party SDKs that collect anything.

## The permissions the app does request

The release build declares four, and none of them can move data off your
device. Listing them here because "no permissions" would be easier to say and
would not be true.

| Permission | What it is for |
| :--- | :--- |
| `RECEIVE_BOOT_COMPLETED` | Android forgets scheduled alarms when the phone restarts. This lets the app put the daily reminder back. |
| `POST_NOTIFICATIONS` | Showing the daily reminder. Android asks you for this at the moment you switch the reminder on, and you can refuse or revoke it. |
| `VIBRATE` | Comes with the notification library, so the reminder can buzz. |
| `com.nttech.loggevity.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION` | Declared by the notification library for its own internal use. Nothing else on the device can hold it. |

The last three arrive from `flutter_local_notifications` rather than being
written by hand, which is exactly why they are worth naming.

## The daily reminder

If you turn it on, the app asks Android to show you one notification a day at a
time you pick. The reminder says nothing about what you have logged. It is
scheduled locally by the operating system, so nothing is sent anywhere and the
app does not need to be running for it to appear.

The reminder is off until you ask for it, and switching it off cancels the
schedule.

## Export and import

**Your data → Export** writes your entries to a JSON or CSV file and hands it
to your device's standard share sheet, so you can send it wherever you like:
email, cloud storage, another app.

Two things worth understanding:

- **Export files are not encrypted.** That is intentional, because an encrypted
  export that only this app could open would be useless as a backup. Treat an
  export the way you would treat any file containing personal health
  information.
- **Once you send an export somewhere, this policy no longer covers it.**
  Whatever service or app you send it to has its own terms.

The file is written to the app's temporary storage on its way to the share
sheet. That storage is private to the app and is excluded from backup along
with everything else.

**Import** reads a file you choose and merges it into your database. An entry
that matches one you already have is skipped rather than duplicated, and a
social check-in for a date you already answered is replaced by the one in the
file. Nothing else is removed. If the file was exported with different scoring
settings, the app tells you and asks before applying them.

## Deleting your data

You are in complete control, and none of it requires asking us:

- Delete individual entries by tapping a category and using the entry list
- Delete everything by uninstalling the app, which removes the database with it
- The **Start fresh** option on the locked-database screen deletes the database
  file outright

Because nothing is ever sent to us, there is nothing for us to delete on your
behalf, and no request process to go through.

## Children

Loggevity is not directed at children and does not knowingly collect anything
from anyone, of any age.

## Health information

Loggevity is not a medical device. It records what you tell it and scores it
against published mortality research. It does not diagnose, treat, or give
medical advice. The reasoning behind every number is written up in the app
under **Settings → About → How scoring works**.

## Changes to this policy

If this policy changes, the updated version will be posted at this address with
a new effective date. Because the app has no way to contact you, checking this
page is the only way to see a change.

## Contact

mattdhoy@gmail.com

---

## How these claims were verified

Included so the claims above can be checked rather than taken on trust, and so
they can be re-checked whenever the app changes.

| Claim | How to verify it |
| :--- | :--- |
| No `INTERNET` permission | `grep uses-permission` on the merged release manifest under `build/app/intermediates/merged_manifest/release/`. The same command produces the four permissions listed above, which is the way to check that list is still complete. |
| Only `RECEIVE_BOOT_COMPLETED` is declared by the app itself | `android/app/src/main/AndroidManifest.xml`. The other three are merged in from the notification library. |
| No backup or device transfer | `android:allowBackup="false"` plus `res/xml/data_extraction_rules.xml` excluding every domain from both `<cloud-backup>` and `<device-transfer>` |
| Database is encrypted | `PRAGMA cipher_version` is checked before the key is applied, in `lib/data/connection.dart`. On the device, the file header is random bytes rather than `SQLite format 3` |
| The reminder is local | `lib/data/daily_reminder.dart` hands one repeating notification to the OS and reads nothing back |
| No analytics or tracking SDKs | The dependency list in `pubspec.yaml`. The packages that reach outside the app's own process are `share_plus` and `file_selector`, which act only when you tap Export or Import, and `flutter_local_notifications`, which talks to the OS notification service and to nothing else |

`test/release_config_test.dart` pins the permission set and the backup settings
on every test run, so a change that adds a permission or re-enables backup
fails the build. It checks the merged release manifest as well as the one a
human edits, which means a permission arriving from a new dependency fails it
too. The merged manifest is build output, so those particular checks are
skipped until `flutter build appbundle` has been run, and the test says so when
it skips.
