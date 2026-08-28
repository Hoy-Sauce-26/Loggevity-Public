# Play Console — Data Safety answers

What to enter in the Data Safety form, and the reasoning for each answer so it
can be defended if Google asks and re-checked when the app changes.

> Drafted against the code, not from assumption. The verification commands are
> in [privacy-policy.md](privacy-policy.md#how-these-claims-were-verified).

---

## The one definition everything turns on

Google defines **collection** as *transmitting data off the user's device*.

Data that a health app stores, processes and displays entirely on the phone is
**not collected** for the purposes of this form. That single definition is why
Loggevity's answers are as short as they are — not because the app handles
little data, but because none of it ever leaves.

---

## Data collection and sharing

| Question | Answer |
| :--- | :--- |
| Does your app collect or share any of the required user data types? | **No** |

Answering **No** here ends the data-type questionnaire. You will not be asked
about Health and fitness, Personal info, or anything else.

**Why this is correct despite the app handling health data:** every entry is
written to an encrypted SQLite database in the app's private storage and read
back by the app itself. There is no server, no account, and no network
permission — the Android release manifest requests **no permissions at all**,
including `INTERNET`, so the operating system will not permit a network
connection.

### Why Export is not "sharing"

Export writes a JSON or CSV file and passes it to the system share sheet.

Google's sharing definition **excludes** "transferring data to a third party
based on a specific user-initiated action, where the user reasonably expects
the data to be shared." Export is exactly that: the user taps Export, chooses a
destination in the OS share sheet, and the data goes where they sent it. The
developer never receives it and no third party is integrated.

Keep this reasoning to hand. It is the one answer a reviewer might query, and
the response is short: the user initiates it, the developer receives nothing.

---

## Security practices

| Question | Answer | Note |
| :--- | :--- | :--- |
| Is all user data encrypted in transit? | **N/A** | Not asked when nothing is collected. If the form presents it anyway, the honest answer is that no data is transmitted. |
| Do you provide a way for users to request that their data be deleted? | **N/A** | Not asked when nothing is collected. Nothing is held anywhere for us to delete. |

Worth saying in the listing description even though the form does not ask: the
user deletes their data by deleting entries, using **Start fresh**, or
uninstalling. It is entirely in their hands and requires no request.

---

## Also required, and easy to miss

### Privacy policy URL — mandatory

Required for **every** app, including those that collect nothing. Needs to be a
public URL, reachable without logging in, and it must stay up.

[privacy-policy.md](privacy-policy.md) is the draft. Two placeholders must be
filled before it goes anywhere: **effective date** and **a contact address you
are willing to publish**. GitHub Pages on this repo is the cheapest hosting
that meets Google's requirements.

### Health apps declaration

Play has a separate declaration for apps handling health or fitness data, and
Loggevity plainly does. Expect it to ask what health data you handle and
whether the app makes medical claims.

**It does not make medical claims, and that should stay true.** The in-app
methodology page opens by saying the score is *not* a risk reduction and closes
by saying it is not medical advice. That framing is a compliance asset as well
as an honesty one — keep it if the listing copy is ever rewritten.

Store listing copy to avoid: anything phrased as diagnosis, treatment,
prevention, or a prediction of lifespan. "Scores your week against published
mortality research" is a description. "Find out how long you'll live" is a
medical claim.

### Account deletion

Not applicable — there are no accounts. The requirement applies to apps that
let users create one.

---

## What would make these answers wrong

Re-check this document if any of the following ever changes, because each one
flips an answer from No to Yes:

- Adding the `INTERNET` permission, or any dependency that requires it
- Adding crash reporting or analytics of any kind, including self-hosted
- Adding sync, accounts, or cloud backup
- Adding any SDK that phones home, however incidentally

`test/release_config_test.dart` fails the build if a permission appears in the
manifest, which catches the first and most of the fourth. It cannot catch a
package that ships its own network code without a manifest change, so a new
dependency is still worth reading.
