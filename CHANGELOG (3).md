# What changed in this version

Notes for whoever picks up the code. Everything below is already in `deploy/`.
The full reference is in `README.md` one level up; this is just what moved.

---

## Site copy and structure were rewritten

The site is now built around one message: a U.S. visitor visa application needs
care and accuracy, and we help applicants prepare properly. The old copy led with
the form; this leads with the offer.

- **Homepage** rewritten. The headline and a Start your application button sit
  above the fold, before any scrolling. New sections on why preparation matters,
  why use us when you could apply alone, the five-stage process, and the current
  environment: interviews now generally required with limited exceptions, and
  applicants generally directed to apply in their country of nationality or
  residence. The FAQ grew from six questions to nine.
- **Six landing pages added**, one per search intent, rather than piling
  everything onto the homepage: `/b1-b2-visitor-visa`, `/ds-160-help`,
  `/visa-interview-preparation`, `/us-visa-renewal`, `/visiting-family-in-usa`,
  `/business-visitor-visa`. All six are in the nav or footer, in the sitemap, and
  have their own title and meta description.
- **About page** rewritten around accuracy, preparation and personal service.
- **Disclaimer strengthened** on every page. It now names the Department of
  Homeland Security and U.S. Embassies and Consulates as well as the State
  Department, states plainly that we are not a law firm and give no legal advice,
  and says applicants may apply directly without us and without paying our fee.
  The footer adds that we cannot guarantee issuance, processing times or
  admission, and that we do not assist anyone in providing false information.

Deliberately not done: no political naming anywhere in the permanent pages. The
copy says "increased screening and changing procedures", which stays true as
administrations change. Run that angle in ads or a dated blog post instead.

---

## The photograph step now offers a choice

The applicant picks: upload it now, or bring a printed photo to the interview.
The step never blocks either way.

Upload-only was losing applications. Almost nobody has a compliant file ready at
the moment they reach that step, and the ones who try mostly fail the 240 KB limit
or the square crop, at the last screen, after half an hour of work.

Bringing a printed photograph is not a workaround, it is the State Department's own
documented route: if the confirmation page shows an X in the photo box, you print
the confirmation as it is and take a compliant 2 by 2 inch (51 by 51 mm)
photograph to the appointment. A failed upload does not mean starting a new DS-160,
and the form now says so in as many words, because applicants who see that X assume
the opposite and start again.

For the server: **`photo` may or may not be in the submit payload.** When the
applicant chose to upload, `photo` is a base64 data URL with `photoName`,
`photoBytes`, `photoWidth` and `photoHeight` beside it, and should be decoded to
file storage rather than left in a database column. When they chose the printed
route, none of those keys are set and `photoMethod` says which route they took.
Read `photoMethod` before looking for a file.

## The last step is now an explicit electronic signature

The button reads **Sign and submit application**, and above it:

> By clicking Sign and submit application you are electronically signing this
> application. Read your answers through once more before you do. An electronic
> signature carries the same weight as a written one.

The certification itself moved to the DS-160's own register: "I certify that I have
read and understood the questions in this application, and that my answers are true
and correct to the best of my knowledge and belief."

**With one thing stated loudly beside it.** A highlighted panel says this is not the
DS-160 signature. The applicant still signs the DS-160 itself on the Department of
State's site, we cannot do that for them, and signing here means confirming their
answers to us so we can prepare and review the application.

Without that panel, the wording implies this page is the government form. It is not,
and every other page on the site says so. There is a test that fails if the panel
disappears.

## The closing message

The confirmation screen now reads: the application is ready to be submitted, and
that does not necessarily mean the nonimmigrant visa application is complete, as
further information may be needed once our team and, in due course, Department of
State personnel have reviewed it.

**One deliberate change from the wording as supplied.** The draft said "after
Visa Journey USA legal associate and Department of State personnel have reviewed".
That is a claim of legal representation, and every other page on this site states
plainly that we are not a law firm and do not provide legal advice. The two cannot
both stand. It reads "our team" instead. If the business does retain a licensed
attorney and wants to say so, that is a different conversation and the disclaimers
have to change with it, not just this sentence.

---

## Tooltips no longer carry the attribution line

"This guidance comes from the Department of State's own DS-160 help text" is gone
from every tooltip. The `official: true` flag on those fields stays, because it is
useful metadata about where the wording came from; it just no longer prints.

---

## Two copies of the form had drifted apart

The form was being worked on in two separate conversations. On September 20, a
fix went into the other copy after `status_error.docx` flagged answers that open
follow-up questions on the real DS-160 but not on ours. That copy never had any of
the work in this package, and this package never had that fix.

This release merges them. From now on, **this package is the only copy.** Any
other `visa-application-form.js` floating around, in an email, a Drive folder or
an older zip, should be treated as out of date.

What came across:

- **Divorced** now opens the Former Spouse questions the DS-160 asks: name, date of
  birth, nationality, place of birth, date of marriage, date it ended, how it ended,
  and where. More than one former marriage can be added. Before, a divorced
  applicant was asked nothing.
- **Widowed** now opens the Deceased Spouse questions: name, date of birth,
  nationality and place of birth, with "Do not know" allowed on the city.
- **Legally separated** now asks about the current spouse. The note on screen
  already told separated applicants those questions were for them, but the status
  list left them out, so the form contradicted itself.
- **A company paying for the trip** is now asked for its name, phone, relationship
  to the applicant and address. Before, choosing "Other company or organization"
  asked nothing at all.

The other-relatives fix in that copy was not carried over. The version here already
matches the DS-160: immediate relatives are listed individually, and "other
relatives" is the single yes or no the official form asks.

`tests/branches.js` holds 24 checks for this class of bug, so the next answer that
opens a follow-up page on the DS-160 has somewhere to be tested.

---

## Why the last release looked unchanged

It was not unchanged. Every change was in the files. The problem was caching, and it
was in the configuration I shipped.

`netlify.toml` cached `site.css` and the JavaScript for an hour, at the same URL on
every deploy. After a redeploy the HTML refreshed straight away, but browsers kept
running the previous `visa-application-form.js` for up to an hour. Nearly every
form change lives in that one file, so the new page shell ran the old form.

**Fixed.** Each asset reference now carries a short hash of the file's contents,
for example `/visa-application-form.js?v=52d38a8525`. A changed file gets a new URL,
so there is no stale copy for a browser to reach for. An unchanged file keeps its
URL and stays cached.

**How to check which release is live.** Every page now carries a build marker. View
source and look near the top for `<meta name="vj-build" content="DATE BUILDID">`.
`BUILD.txt` at the site root lists the same id and each asset's hash. If the id on
the live page does not match `BUILD.txt` in the folder that was uploaded, the new
deploy has not gone out.

If a browser was already holding the old file before this fix, one hard refresh
clears it: Ctrl+Shift+R on Windows, Cmd+Shift+R on a Mac. After this release that
should never be needed again.

---

## Read this first: nothing sends email yet

`DEMO = true` in `application.html` and `resume.html`. While it is set, no request
leaves the browser. Reference numbers are generated locally and nothing is stored.

That flag used to hide a worse problem. The email callbacks resolved successfully
in demo mode, so the form displayed **"Sent to you@example.com"** for an email that
never left. If the site had gone live with `DEMO` still set, an applicant would have
closed the tab believing their reference number was in their inbox.

Demo mode now fails on purpose and says so on screen: "Demo mode: no email was
actually sent. Wire the email endpoint on the server to enable this." Same for the
resume link behind "Save and finish later", which had the same fault. The
single-file preview says "This is a preview, so no email was actually sent."

To make it real, flip `DEMO` to `false` and build:

    POST /api/application/email-reference
      in:   { applicationId, email, kind: "reference" }
      out:  200

Two rules, both non-negotiable:

- **The message contains the reference number and nothing else.** Never the
  security answer. Those two together are the entire credential, and the screen
  directly above the button tells applicants not to send them in one message.
- **Rate limit it.** An open send endpoint keyed on an application id is a way to
  post mail to strangers.

Errors can now carry a `userMessage` property, and the form shows it verbatim. Use
it so a real failure reads "That address was rejected by our mail provider" rather
than a generic apology. Without it, the generic message is used.

The other endpoints are specced in `README.md`. `GET /api/application/:id` is the
one that makes a browser refresh survive a closed tab, and it must authorise on the
session set at create or resume, never on the reference number alone. The reference
is printed on screen and emailed; it is not a secret.

---

## Date entry was rebuilt

Native `<input type="date">` is gone from the form. It was the wrong control for a
date of birth: the picker opens on the current month and expects the applicant to
scroll back forty years, and on several browsers the year segment resists being
typed into once a value is set. Applicants could not reach their own birth year.

Every date is now three controls, day and month as dropdowns and the year as a
typed four-digit box. That is what the DS-160 itself uses. On a phone the year box
brings up the number keypad and the month drops onto its own line.

**Nothing changed behind the scenes.** The value is still a plain `yyyy-mm-dd`
string under the same key, so saving, the review screen, the draft cache and the
submit payload are untouched. Half-entered dates are held in the widget rather than
the answer data, so a lone month never reaches the server or counts as an answer,
but it stays on screen while the applicant finishes typing.

Validation improved as a side effect, since the parts are now ours:

- Incomplete: "Enter the day, the month and the year."
- 31 February and similar: "That date does not exist. Check the day and the month."
- A typo like 1084: "Check the year."
- A future date of birth is still caught as before.

Applied to all 18 date fields. Where the form asks for a year on its own,
`type: "year"` gives the same typed box rather than a number spinner whose arrows
step one year at a time. Use it for any year field added later.

---

## Fixes that came out of walking real applicants through the form

Four awkward applicants were run end to end: an Algerian with dual French
nationality, a single-name Indonesian worker applying from a third country, a
six-year-old whose mother was typing, and a stateless applicant on a travel
document. These are now permanent test suites, because this is the category of bug
a form written from a US or western European default keeps producing.

**Accented names were rejected with the wrong reason.** Aïcha and Benoît failed
with "Numbers are not accepted here", which is both wrong and useless. They are
still rejected, since the DS-160 wants English characters, but the message now
names accents as the cause and points at the machine-readable strip at the bottom
of the passport photo page, which holds the exact spelling to type. Non-Latin
script gets a third message pointing at the native-alphabet field. Names beginning
with an apostrophe, like 'Abd Allah, were rejected outright and now pass.

**US territories were missing from the state lists.** Guam, the US Virgin Islands,
American Samoa and the Northern Mariana Islands. An applicant whose US contact
lives in any of them could not complete the step. All added.

**Only one of eight phone fields mentioned the country dialling code.** All of them
do now, and the US contact field says to use a US number.

**Children could not honestly certify.** The confirmation read "I confirm that the
answers above are true and correct", which a six-year-old cannot agree to and which
left the parent unsure whose name they had put to it. The review step now works the
age out from the date of birth: under 16 a parent or guardian signs, at 16 or 17
the applicant signs but a parent may help. The certification wording changes to
match, and it reminds them to name themselves in the "who helped you" section.
Consent text may now be a function of the answers, which is how that switch works.

**Dual nationals were not told they may not need a visa at all.** When a second
passport is from a Visa Waiver country, the form now says an ESTA may be possible
instead, with no interview, no MRV fee and no fee to us, and points at
travel.state.gov. That list changes: re-check it yearly. The comment in the code
says the same. This one is a deliberate business decision, not just a fix.

**Email confirmation was a text field**, so phones offered the wrong keyboard. Now
an email field.

---

## Tests

330 checks across seven suites in `tests/`. Run them before and after any change;
`tests/README.md` has the commands. Only dependency is `npm install jsdom`.

Start with `rebuild_check.py`. It copies the package to a temp directory, deletes
everything generated, runs all three build scripts and asserts it all comes back
byte-identical. It found two real bugs the day it was written, including a build
script that had silently stopped generating the two legal pages while stale copies
sat in `src/` masking it.

---

## Two things not to undo

Both are commented in the files. Both look like tidy-up candidates to anyone who
was not here, and both have been reintroduced once already.

**In `site.css`:** the button rules carry a `.vj-site` prefix. Most buttons on this
site are anchors, and `.vj-site a` sets a link colour at a higher specificity than a
bare `.btn-primary`. Strip the prefix and the filled buttons go back to dark teal
text on a teal background.

**In `visa-application-form.js`:** free-text, date and number inputs write through
`setSmart`, which repaints only when the step's visible or required fields actually
change. Repainting on every keystroke or on blur destroys the node mid-gesture,
which broke typing, Tab, clicking from one field into another, and date entry
outright. `render()` also wraps `paint()` to restore focus and caret for the
repaints that do still happen. Keep both.
