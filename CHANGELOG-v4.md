# The second factor is built

The admin sign in is now two steps: email and password, then a six-digit code
from an authenticator app. This was the one thing standing between the package
and a card-reveal route that could go live, because PCI DSS requires a second
factor for every access to card data and this login can reveal card numbers.

**What changed on the front end.** `admin.js` gained a code step and a second
callback, `onVerifyCode({ code })`. The two steps are deliberately separate
calls: the password goes up once, and the code step sends only the code. The
browser never holds the password between them and never sends it twice.

A server that has no second factor enrolled yet answers the password step with
`{ email, name }` and the page signs straight in, exactly as before. So the
password step can ship on its own and everything except the reveal route works
while TOTP is still being built. Answer `{ mfaRequired: true }` instead and the
code step appears.

**What the server owes.** `POST /api/admin/session` now answers the password
step with `{ mfaRequired: true }` and a short-lived half session cookie, and a
new `POST /api/admin/session/verify` takes the code and upgrades that to a real
session with `mfa_passed` set. The card-reveal route checks that flag. Full
instructions, including generating the secret, the `otpauth://` URI, the drift
window, replay prevention and recovery codes, are in README.md under "The second
factor".

**Recovery codes matter more here than usual.** There is one account. Losing the
phone without them locks the only login out of the site with no way back. Ten
codes at enrolment, hashed like the password, stored somewhere that is not the
same phone.

**In the preview**, the code is `123456`, and it says so when you get it wrong.
That is not a second factor and is not pretending to be one: a static file has
no server and no secret. It exists so the two-step screen can be clicked through
and reviewed. `DEMO_MFA_CODE` and `demoVerifyCode` come out with `demoSignIn`
and the DEMO flag before go-live, and the go-live list in README.md now names
them.

Tests are at 680 across eleven suites, 211 of them on the admin screen, with a
new section covering the two steps: that a wrong password never reaches the code
step, that nothing is signed in until the code is accepted, that the password is
not sent a second time, and that a server with no second factor still signs
straight in.

---

# What changed in this version

Notes for whoever picks up the code. Everything below is already in `deploy/`.
The full reference is in `README.md` one level up; this is just what moved.

---

## One admin account

The admin screen now has exactly one login, `josephschulman2013@gmail.com`, in
place of the open-ended "managers" it was designed around.

**On the server** the account is two environment variables, `ADMIN_EMAIL` and
`ADMIN_PASSWORD_HASH` (a bcrypt hash, cost 12). Sign in compares the email
case-insensitively and the password with bcrypt's constant-time compare, answers
a wrong email and a wrong password identically, rate limits and locks out, and
issues a Secure, HttpOnly, SameSite=Strict session cookie. There is no server
code in this package, so this is the `onSignIn` contract at the top of
`admin.js`, the new "The admin account" section of `README.md` (with the one
command that makes a new hash) and a new section in `deploy/DEPLOY.md`. The
plain password is not in any file here.

**In the screen** the heading is "Admin sign in", the bar reads "Signed in as"
and the account email, and notes, activity and "charged by" are written by
"Admin". `onSignIn` now returns `{ email, name }`. The "Last opened by" column
is gone, since with one login it would always say the same thing, and
`lastOpenedBy` is out of the `onList` contract. `mount()` takes
`admin: { email, name }` where it took `manager`.

**In the previews** (`admin.html` while `DEMO` is true, and
`preview/visa-journey-admin.html`) only the admin email gets in, and the password
is checked by comparing a SHA-256 of what was typed, via WebCrypto, with a hash
in the page. That is for the static preview only: the hash is readable by anyone
with the file, so it is marked to be deleted with the `DEMO` flag before
go-live. The yellow notice now says "Sign in with the admin account". The demo
notes and activity, which named two invented managers, now say "Admin".

**Two things to do about it.** Turn on a second factor (TOTP) for this account:
it can reveal full card numbers. And only one person should hold it, because the
reveal log can then say which account viewed a number but not which person.
Both are explained in the README.

Test count is 658. `admin.js` gained 24: wrong email, wrong password, empty
password, matching refusals, the right account in any case, the bar showing the
account email, "Admin" on new notes, no "last opened by" column, no WebCrypto,
and the production sign in's 401 and 429 handling. The tests never type the
password; the success path stubs the hash compare.

---

## The admin can now see the full card number

The fee is now charged later, by hand, on a processor the site is not connected
to, so the manager needs the actual card number rather than a saved token. This
was chosen knowingly. It puts the site in PCI DSS SAQ D, and the server side
must not be built until `README.md`, "Storing full card numbers", has been read.

**The card step** (`payment-step.js`, new `collectCard` mode, now on in
`payment.html`) takes name on card, card number, expiry month and year, and an
optional billing postal code. It checks Luhn, length and brand (Visa, Mastercard
including the 2-series, Discover, Amex) and refuses an expired card, all before
anything is sent. The number is grouped in fours as it is typed and the brand
lights up. There is no CVV field, on purpose. The details go to `onSaveCard` and
are held in memory only; they are never written to browser storage, and the card
field names are now on the form's `NEVER_CACHED` list too. "Nothing is charged
today" and the rest of the charge-later wording are unchanged. "Card details are
never stored on our servers" is gone, because it stopped being true. The privacy
policy line about card numbers is rewritten for the same reason.

**The admin Payment panel** shows the card masked by default, "Visa •••• •••• ••••
4242, expires 09/28". **Show full number** calls the new `onRevealCard` and shows
the number in fours in a monospace font with a Copy button, plus the name, expiry
and postal code. It hides itself after 60 seconds, and on Hide, navigation, sign
out, printing and closing the page. The number is never kept in the screen's
state: once hidden it is gone, and seeing it again is another reveal. Each reveal
adds "Viewed full card number" to the Activity list, and the server logs it too.
Print styles never show it.

**Mark as charged** replaces the Charge button: amount, date and the processor's
reference. Recording it marks the application paid and the server purges the
stored number. `onCharge` still works if it is supplied, for when the card goes
back to a saved token. The expired-card warning is unchanged.

**Demo data** carries the networks' test numbers (4242 4242 4242 4242, 5555 5555
5555 4444), so the reveal can be clicked through in both previews. The paid
application's number shows as already purged.

Test count is 634 across eleven suites. `admin.js` gained 54: the masked line,
the reveal, the copy, the auto-hide and every other wipe, a late reveal being
dropped, the Luhn and expiry checks on the card step, nothing reaching browser
storage, and `onCharge` still working when it is supplied.

---

## The fee is now taken after the work, not before

The card step no longer charges. The applicant authorises a card at the end of
the form and is told plainly that nothing is taken today; a manager charges the
$185 from `/admin` once the application has been prepared and reviewed. Set
`chargeLater: false` in `payment.html` to go back to charging upfront.

In the admin, the Payment panel shows "Visa ending 4242, expires 09/28" and a
Charge $185 button. One click, after a confirmation, and the fee is taken. Once
it is paid the panel shows the amount, the date, who took it and a receipt link,
and the button is gone.

### What is stored

The provider's customer id and payment method id, plus brand, last four and
expiry. Nothing else.

Not the card number, and not the CVV. The CVV one is not a judgement call: PCI
DSS prohibits keeping it after authorisation, with no exception for encrypting
it. The number is permitted with the full control set, but taking it would move
this site from the short self-assessment it qualifies for today to the long one
with quarterly scans and key management, on a database that also holds passport
and national ID numbers. The saved-token approach avoids that and gives the
manager a better screen than a number on a page would.

### Two endpoints to build

`POST /api/application/setup-intent` returns a Checkout Session URL in **setup**
mode, not payment mode. `POST /api/admin/applications/:id/charge` takes the
money. Both are specified in README.md, along with three rules for the charge
endpoint: the amount comes from the server, it is idempotent keyed on the
application, and it refuses an application that is already paid. The admin
disables its button on the first click, but that does not survive a flaky
connection, and a retry after a timeout takes the money twice if the server is
not keyed properly.

### Two things that go wrong on this timeline, built for

An application submitted in August with an interview in November leaves a card
authorised for months. Both of these are ordinary here rather than edge cases.

The card expires first. The admin checks the stored expiry and warns before the
button is pressed rather than letting the manager find out through a decline.
There is a demo application in exactly this state.

The bank asks the applicant to confirm. An off-session charge can come back
needing authentication, which is likelier with cards issued outside the US, and
most of these applicants are. The admin reports that differently from a decline
and says to send a payment link, since nothing the manager does alone will fix
it. Worth checking India specifically before launching there: RBI rules require
network tokenisation and limit what a merchant may keep.

Test count is 580 across eleven suites.

---

## The manager admin screen is built

`/admin`. Sign in, a searchable list, and a detail view of every answer. The
front end is real; the six endpoints behind it are not, so `admin.html` ships
with `var DEMO = true` and three invented applications, exactly the way
`application.html` and `payment.html` already work. Flip the flag and the fetch
calls beside it take over. What each endpoint owes back is at the top of
`admin.js`.

To look at it without a server, open `preview/visa-journey-admin.html`. Sign in
with anything. The applications, names and passport numbers in it are made up.

### The detail view walks the form's schema

It renders from `VJ_SCHEMA.STEPS` rather than from the answer object, and the
difference is not cosmetic:

- The manager reads the exact wording the applicant answered. Change a question
  in the form and this follows, with nothing to keep in sync and no second copy
  of two hundred labels to drift.
- A question that was asked and left blank still appears, marked unanswered.
  Walking the answers would silently drop every unanswered question, which is the
  single thing a reviewer most needs to see.
- A question this applicant was never shown is left out, because `showIf` is
  evaluated against their own data. An unasked question is not an unanswered one.

The DS-160 country and occupation codes sit beside those answers, since the
manager is retyping all of it into the Department of State's own form.

Worth knowing if you are tempted to reuse the form's own `renderReview`: do not.
It is private to that module, derives labels from the key name rather than the
schema label, and deliberately drops empty values and booleans. Right for an
applicant confirming their work, wrong for a manager reviewing it.

### What it will not do

It cannot edit an applicant's answers, and that is deliberate. They signed them
electronically. A manager quietly changing them afterwards would make that
signature meaningless. Corrections go back through the applicant or become a
note. The suite asserts the detail view has no editable fields beyond the status
control, the note box and the search.

### Keeping it off the public site

Not in the nav, not in the footer, not in the sitemap, `Disallow: /admin` in
robots.txt, `noindex,nofollow` on the page and `X-Robots-Tag` plus `no-store` in
the host config. `deploy_checks.py` asserts that no public page links to it.

None of that is access control. The sign in is, and every admin endpoint has to
check the manager session server side. A route that works because nobody has
found the URL is the usual way this kind of data leaks.

### Two bugs found while building it

A literal `</script>` inside `admin.js`'s own usage comment ended the script tag
early when the file was inlined into the single-file preview, which blanked the
page with a syntax error and no other clue. `bundle.py` now escapes that
sequence in every inlined file and asserts the finished preview has exactly three
script blocks.

And the first version of the admin test suite asserted against
`document.body.textContent`, which includes the text of every `<script>` on the
page. The demo applications live in one of those, so several checks were reading
the source data rather than the rendered page and would have passed even if
nothing rendered. Every content assertion now reads the mount container.

Test count is 546 across eleven suites, all passing from a clean extraction.

---

## If you are not deploying to Netlify

`netlify.toml` is read by Netlify and ignored by every other host. It was doing
three jobs, and one of them is load-bearing:

**Clean URLs, which the site cannot run without.** Every link in the HTML points at
`/about-us`, not `/about-us.html`. If the server does not map an extensionless path
to the `.html` file on disk, every link in the site 404s. Most static hosts already
do this, so if `/about-us` is loading for you today, yours does.

**Security headers.** Five of them. Nothing breaks without them; a security review
will ask.

**301s from the old `.html` paths**, so one page is not indexable at two URLs.

`deploy/server-config/` now has all three written out for Apache, nginx, IIS,
Vercel and Cloudflare Pages, with a README saying which file to rename and where to
put it. Copy the one that matches. Nothing picks them up on its own.

One bug worth knowing about if you were going to write these rules yourself: the
obvious strip-the-extension rule turns `/index.html` into `/index`, which then
serves the homepage at a second URL, which is the thing the rule was added to
prevent. Each config here handles `index.html` first, and `rebuild_check.py` now
asserts the ordering so it cannot regress.

`live_check.py` also checks all three on whatever host you use, so you are not
taking the config on trust:

    python3 tests/live_check.py https://www.visajourneyusa.com

If your host genuinely cannot serve extensionless URLs, say so rather than
hand-editing the HTML. There are over five hundred internal links across fourteen
pages and the next build would overwrite the edits. Regenerating with `.html` on
every link is a one-line change to `deploy.py`.

---

## The 404s on the live site: six pages are not on the server

Read this one first. There is nothing to fix in the code.

Six links in the navigation and footer of **every** page on the live site return
404 right now:

| URL | In the nav | In the footer |
| --- | --- | --- |
| `/b1-b2-visitor-visa` | yes | yes |
| `/ds-160-help` | yes | yes |
| `/visa-interview-preparation` | yes | yes |
| `/us-visa-renewal` | — | yes |
| `/visiting-family-in-usa` | — | yes |
| `/business-visitor-visa` | — | yes |

The eight other pages are live and correct: `/`, `/about-us`, `/contact-us`,
`/application`, `/payment`, `/resume`, `/terms-and-conditions`, `/privacy-policy`.

All six missing pages are in `deploy/`, have been in every handover zip, are in
the sitemap, and pass every check locally. They were never uploaded. The build
marker on the live pages reads `c1184e3d7b`, which is this build — so what is on
the server came from the right folder, just not all of it.

**The fix is to upload the whole `deploy/` folder again,** as a folder, not a
selection of files from it. No rebuild is needed and the build id does not change,
because none of the site content has changed. If you are deploying by drag and
drop, drop the folder itself onto Netlify rather than opening it and selecting
files; selecting files loses the ones you did not click.

Then confirm it:

    python3 tests/live_check.py https://www.visajourneyusa.com

That requests every URL this folder is meant to serve, compares the live build
marker against `BUILD.txt`, and follows the links on each live page. It exits 0
only when everything answers 200. Run it after every deploy from now on — it is
the one check that tests the server rather than the build, and the server is what
went wrong.

### Why no test caught this

They all ran against the folder, and the folder was right. Three checks have been
added so the gap is covered from both ends:

- `tests/link_check.py` resolves every link in all three builds against what will
  actually be served, including the navigation performed from inline scripts and
  the hash routes in the single-file preview. 576 links in the deploy build, 509 in
  the source build, 60 in the preview; all resolve.
- `tests/crawl.py` requests every URL for real and follows links out from `/`, so a
  flaw in that reasoning shows up as a 404. It also reports any page in the folder
  that nothing links to.
- `tests/live_check.py` is the one above, and the only one that looks at production.

Two smaller things fixed alongside them: `tests/serve_clean.py` was answering
unknown paths with a 200 and the 404 page's body, which is what Netlify does *not*
do and which would have hidden a broken link from any crawler — it now sends a real
404 status. And `DEPLOY.md` still said the folder held "six pages plus a 404" from
before the landing pages existed; it now lists all fourteen and says plainly that a
partial upload produces 404s on a working build.

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
