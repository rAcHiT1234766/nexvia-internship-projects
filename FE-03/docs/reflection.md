# Individual reflection

## 1. Which screens, persistence functions, sync behaviors, tests and deployment tasks did I own?

I built FieldLog for internship assignment FE-03 with an AI coding assistant. The
assistant wrote most of the code, the automated tests and the documents from the FE-03 brief. My
part was directing the work, making the product decisions, setting up and running the real
services, and testing the app myself on a real phone.

Things I did myself:

- **Deployment.** I ran the production deploys to Firebase Hosting myself, after the assistant
  had checked the build. The app is live at https://fe-03-56215.web.app.
- **Design direction.** After the first redesign I said it looked generic, like an "AI-made" app,
  and the sign-in screen most of all. I asked for something with more character. That became the
  "field job book" style: a dark app bar, a paper-coloured background, safety orange as the only
  signal colour, and IBM Plex fonts.
- **Real-device testing** on my Android phone in Chrome, on 31 October and 1 November:
  - I installed the app from the live site. It opens from the home-screen icon as "FieldLog", full
    screen with no address bar, on the Jobs list (T42).
  - I did the airplane-mode test. I edited a job and created a new one offline, closed and
    reopened the app while still offline, then turned the network back on and checked on my PC
    that both changes had synced.
  - I tested push notifications: a banner inside the app while it was open (T43), a phone
    notification while it was closed (T44), and tapping it opened the job (T45). For this I signed
    in with gcloud, found my user ID, device ID and job ID in the Firestore console, and ran the
    test-push script.
- **GitHub.** I created the repository, and the code was pushed there. The automated checks
  (GitHub Actions) passed on the first run.

The screens are Sign-in, Jobs list, Job details, Job editor, Sync center and Settings. The
persistence and sync code is in `src/jobs.ts`, `src/db.ts` and `src/sync/`.

I set up the Firebase project myself, on my own: Authentication, Firestore, Cloud Messaging and
the VAPID key. The part of the app I checked and asked to change myself was the overall design.

## 2. What did I initially misunderstand about service workers versus IndexedDB?

I did not have a real misunderstanding here: I already knew the difference between service
workers and IndexedDB before this project.

How the two split the work in FieldLog: they do two different jobs, and an offline app needs both.

- The **service worker** (Workbox, `src/sw.ts`) stores the **app itself**: the HTML, JavaScript,
  CSS and fonts, 26 files in total. That is why FieldLog *opens* with no internet. It never
  caches job data or Firebase traffic.
- **IndexedDB** (through Dexie, `src/db.ts`) stores the **data**: the jobs, the outbox of changes
  not sent yet, and conflict notices. That is why my jobs are still *there* with no internet.
- Without the service worker, the app would not load offline. Without IndexedDB, it would load
  but be empty.
- The service worker also does not sync my data in the background. FieldLog only syncs while it
  is open, or when it is opened again.

## 3. Why did I choose a custom durable outbox instead of relying only on Firebase offline persistence?

Firebase can queue writes offline by itself, but FieldLog deliberately does not use that.
Firestore runs with a memory cache only, and every change goes through our own queue (the
"outbox") in IndexedDB. The brief asked for an explicit, durable queue, and the assistant designed
it that way. The reasons it makes sense:

1. **I can see the queue.** Firebase's queue is hidden. With our own outbox, the Sync center can
   show "2 changes waiting to sync", which change failed and why, and what needs attention.
2. **Retries are harmless.** Every change gets its own ID (`opId`). The server saves a receipt
   for it in the same transaction as the change, so if a change is sent twice, the second time
   does nothing.
3. **Conflicts follow a rule I chose**, not just "the last write overwrites" (see question 5).
4. **Nothing is lost on reload or restart.** A change is saved locally together with its outbox
   entry in one transaction, so either both are stored or neither is.

The reason that makes the most sense to me is the first one: I can see the queue. I already knew
Firebase had its own offline mode before this project.

## 4. What is the difference between at-least-once delivery and idempotent state changes?

**At-least-once delivery** means the app keeps sending a change until it is sure the server got
it. It will never give up silently, but sometimes it sends the same change twice.

**Idempotent** means doing the same change twice has the same result as doing it once.

An example from my own job: I mark "Boiler Annual Service" as completed. The server saves it, but
the connection drops before my phone hears back. My phone does not know it worked, so it sends
the change again. Without protection, this could count twice, or overwrite something another
device changed in between. In FieldLog, the server first looks for a receipt with that change's
`opId`. It finds one, so it just answers "already done" and changes nothing.

So delivery is at least once, but the *effect* happens exactly once. That only holds as long as
the receipts are kept, and they are never deleted. Tests T08 (the same change delivered twice),
T09 (lost response) and T21 (tab closed in the middle of sending) cover this.

This part was not confusing for me; it made sense straight away.

## 5. Which conflict policy did I implement, and what does it still not solve?

FieldLog uses a **field-level merge**. Every change remembers three versions of each field:
the value it started from, the user's new value, and the server's current value.

- If two devices change **different fields** of the same job (one the status, the other the
  notes), both changes are kept.
- If they change the **same field**, the change the server saved last wins. "Last" is decided by
  the job's revision number on the server, not by the phone's clock, because phone clocks can be
  wrong. Test T29 shows that a device with its clock set a year ahead does not win.
- The losing value is not thrown away. A conflict notice appears with a "Use this value" button.
- **Deleting is final.** If one device deletes a job while another edits it, the delete stays.
  The edits are kept in a notice and can be saved as a new job.

What it does not solve:

- The notes are one field. If two people edit different lines of the notes, that still counts as
  a conflict, and one version wins.
- If a value is changed and then changed back on another device, the merge sees no change.
- The merge runs on the phone. The server rules check the structure and the owner, not the merge
  itself. That is fine for one person's jobs, but not for a team.
- It is not real-time collaboration like Google Docs.

I think "the later server save wins, with a notice" is the right rule for technicians, and I
would keep it. It is simple, and because the other value is kept in a notice, nothing is lost
silently.

## 6. Which failure case forced a change in my design?

The biggest one came from the emulator test where two devices save the same job at the same
moment. One device should simply retry. Instead, its save came back as **"permission denied"**.
Firestore checked the security rule "the revision must go up by exactly one" against the revision
the *other* device had just saved, so the slower save failed the rule. The app then treated it as
a permanent rejection and blocked the change.

The fix: when a save fails with "permission denied", the transaction is run again, up to 3 times.
This is safe because a failed save changes nothing, the receipt check stops anything happening
twice, and a real permission problem fails every time anyway. When the fix was removed on
purpose, the test failed again, which proved the test really catches the problem.

A smaller one I hit myself: my first two test pushes showed nothing on my phone. Firebase had
accepted them, but they had no job ID, and the app drops messages without one. We changed the
test script so it refuses to send without `--job`.

Of all the failures, the "permission denied" race mattered most to me.

## 7. What did the browser/device tests reveal that unit tests did not?

The unit tests run the logic with a fake server, so they could not catch these:

- **The update reload did not happen (T38).** If a page was first opened before the service
  worker controlled it, tapping "Update now" activated the new version but did not reload the
  page, so the old code kept running. Fixed in `src/pwa.ts`: the app now reloads exactly once.
- **The "ready to work offline" notice moved the page**, which Lighthouse flagged as layout shift.
  The first fix then made the notice cover the Save button on short screens. Notices now sit at
  the bottom without covering anything.
- **Screen readers announced messages twice**, because a message was in both a visible banner
  and a hidden announcer.
- **The Lighthouse mobile score was wrong at first**, because our local test server did not
  compress files the way Firebase Hosting does.

And on my real phone:

- **Test pushes without a job ID were silently dropped.** The automated tests only ever sent valid
  messages, so they never showed this.
- Installing the app and using it in airplane mode confirmed on a real device what the browser
  tests could only simulate.

What surprised me most on my phone was that airplane mode really worked: my edits survived
closing the app while offline, and they synced after I reconnected.

## 8. How did I use the AI assistant, and how did I verify the code it produced?

I used an AI coding assistant for most of the project. It planned the work from the FE-03 brief, wrote the
code, the automated tests and the documents, ran the tests, and explained things when I asked.
I gave the instructions, made the decisions about what I wanted, and did the real-world checks.

How the work was checked:

- **Automated tests:** 83 unit tests, 21 security-rule tests, 17 sync tests with two simulated
  devices, and 14 browser tests on the production build. They all pass, and they also passed on
  GitHub Actions from a clean install.
- **Mutation checks:** for important fixes, the assistant broke the code on purpose to show the
  test then fails. A test that cannot fail proves nothing.
- **My own checks on a real phone:** install, airplane mode and push, as in question 1.
- **Checking the real data:** I looked in the Firestore console and saw my job saved with its
  revision number, and my phone registered for notifications.
- **Looking at the design myself:** I rejected the first redesign because it looked generic.

I read most of the code myself. The clearest time I disagreed with the assistant was the design:
I rejected the first redesign because it looked generic, and asked for something with more
character.

## 9. Which optional feature did I intentionally leave out to keep the required project reliable?

The brief lists several optional features. These were left out on purpose:

- **Incremental sync** (only downloading jobs changed since last time). Every sync re-reads all
  jobs instead. It is simpler and cannot miss a change. The brief itself warns that a half-correct
  incremental sync can skip records. It is fine for hundreds of jobs, not thousands.
- **Background sync while the app is closed.** Sync only happens while the app is open.
- **Photo attachments.**
- **Automatic push when a job changes.** Push is sent by a test script, not by a server trigger.
- **Merging notes line by line.**

The one I would name as left out on purpose is photo attachments. They are not needed to show
that offline saving and sync work, so leaving them out kept the focus on the required parts.

## 10. What would I change before using this application for a real field-service team?

Possible changes, from the project's own limits list:

- **Teams and shared jobs.** Right now each account only sees its own jobs.
- **Run the merge on a trusted server** (for example a Cloud Function), so the server enforces
  the conflict rule, not each phone.
- **Incremental sync**, so it stays fast with thousands of jobs.
- **Background sync**, so changes upload even when the app is closed.
- **Photo attachments**, for proof of work.
- **Automatic notifications** when a dispatcher changes a job.
- **Testing on iPhone/Safari and Firefox**, and with a real screen reader.
- **Password reset** on the sign-in screen.

I would do three of these first: teams and shared jobs, because right now each account only sees
its own jobs; photo attachments, as proof of work; and running the merge on a trusted server, so
the server enforces the conflict rule instead of each phone.
