# Privacy Policy — Loggevity

**Effective date:** Aug 28 2026
**Contact:** mattdhoy@gmail.com

> **This is a draft, not legal advice.** Every factual claim in it was checked
> against the code and the built release artifact (see "How these claims were
> verified"), but whether it satisfies your obligations in your jurisdiction
> is a question for a lawyer. Have someone qualified read it before you
> publish it as the policy of record.

---

## The short version

Loggevity does not collect anything. There is no account, no server, and no
analytics. Everything you log stays in an encrypted database on your own
device, and the only way any of it leaves is if you choose to export it.

---

## What Loggevity stores

Everything you enter into the app:

- Activity you log — minutes of exercise, hours of sleep, hours of company,
  time outdoors, and the dates and times you recorded them
- Your answers to the weekly social check-in, if you use that mode
- Your settings — which day your week starts, your socializing target, and any
  category weights you have changed
- Scores for completed weeks, calculated from the above

That is the complete list.

## Where it is stored

In a single database file inside the app's private storage on your device.

That file is encrypted with SQLCipher. The encryption key is generated on your
device the first time you open the app and is held in the Android Keystore. The
key never leaves your device and is not included in any backup.

**Loggevity is excluded from Android's automatic backup and from
device-to-device transfer.** This is deliberate. Because the encryption key
cannot leave your device, a copy of the database restored onto a different
phone could not be read — so rather than hand you a file nothing can open, the
app does not back it up at all. If you want your history to survive a new
phone, use **Your data → Export** and keep the file somewhere you control.

## What Loggevity sends

Nothing.

The Android app requests **no permissions at all**. It does not hold the
`INTERNET` permission, which means the operating system will not allow it to
make a network connection even if it tried to. There is no server for it to
talk to.

There are no analytics, no advertising, no crash reporting, no tracking
identifiers, and no third-party SDKs that collect anything.

## Export and import

**Your data → Export** writes your entries to a JSON or CSV file and hands it
to your device's standard share sheet, so you can send it wherever you like —
email, cloud storage, another app.

Two things worth understanding:

- **Export files are not encrypted.** That is intentional: an encrypted export
  that only this app could open would be useless as a backup. Treat an export
  the way you would treat any file containing personal health information.
- **Once you send an export somewhere, this policy no longer covers it.**
  Whatever service or app you send it to has its own terms.

**Import** reads a file you choose and merges it into your database. Import
never deletes anything already there.

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
against published mortality research; it does not diagnose, treat, or give
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
| No permissions, no `INTERNET` | `grep uses-permission` on the merged release manifest under `build/app/intermediates/merged_manifest/release/` |
| No backup or device transfer | `android:allowBackup="false"` plus `res/xml/data_extraction_rules.xml` excluding every domain from both `<cloud-backup>` and `<device-transfer>` |
| Database is encrypted | `PRAGMA cipher_version` is checked before the key is applied — see `lib/data/connection.dart`; on-device the file header is random bytes rather than `SQLite format 3` |
| No analytics or tracking SDKs | The dependency list in `pubspec.yaml` — the only packages that touch the outside world are `share_plus` and `file_selector`, both of which act only when you tap Export or Import |

`test/release_config_test.dart` pins the first two on every test run, so a
change that silently adds a permission or re-enables backup fails the build.
