---
name: client-recording-uploads
description: >-
  Files Teams meeting transcripts and recordings into each client's SharePoint
  folders on one recurring schedule. A one-time setup per client saves a profile
  (meeting keywords, client email domains, destination folders); one scheduled
  batch run then covers every saved client: it finds each client's meetings on
  the calendar, classifies each internal or external, saves each transcript as a
  .vtt file, copies real video recordings, and emails a summary. Use when the
  user says "set up recording uploads for [client]", "add [client] to recording
  uploads", "reconfigure recording uploads for [client]", "run recording uploads
  now", or "dry run recording uploads". Do NOT use for a one-off copy of a single
  file, meeting summaries or notes (use meeting-intel), or deleting files.
cowork:
  category: automation
  icon: CloudArrowUp
---

# Client Recording Uploads

Files each client's Teams meeting transcripts and recordings into that client's SharePoint folders, split internal vs external, on **one scheduled batch run** that covers every client with a saved profile. Every per-client setting is **confirmed with the user at setup and saved to a profile**, never inferred at run time.

## When NOT to Use

- A one-off copy of a single file → copy it directly, no profile or schedule.
- Meeting summaries, notes, decisions, or action items → use meeting-intel.
- Deleting or moving files already in SharePoint → manual; these tools copy and upload only.

## How This Runs

| Mode | Triggered by | Covers | Writes files? |
|---|---|---|---|
| **Setup** | "set up / add / reconfigure recording uploads for [client]" | One client | Saves the profile only |
| **Dry run** | End of every setup, or "dry run recording uploads" | One client or all | No. Lists what *would* be filed |
| **Batch run** | The one recurring scheduled task | Every client with a profile | Yes |
| **On-demand run** | "run recording uploads now [for client] [past N days]" | One client or all | Yes |

There is **one** scheduled task, not one per client. The calendar is pulled once per run and filtered per client, so adding a client means adding a profile, never another task.

---

## Setup (one client)

1. **Start from what exists.** If `clients/<client-slug>.md` exists, this is a reconfigure: load it and treat its values as proposals. Call `GetScheduledPrompts`; if an older per-client recording-upload task for this client exists, read its description and propose its values too.
2. **Confirm every profile field** with `AskUserQuestion`, in batches of at most four questions, recommended option first. For anything the user must type or paste (folder links, domains, keywords), pass `options: []` for a free-text answer rather than authoring an "Other" option.
   - **Client name**, used in file names.
   - **Keywords**: one or more words or phrases that mark a meeting title as this client's.
   - **Client domains**: every email domain that makes a meeting external for this client. Include outside advisors or a second company domain when they belong to the engagement.
   - **Firm domain** (normally `eidebailly.com`).
   - **External folder**: the SharePoint link for external meetings.
   - **Internal handling**: skip internal meetings, file them in the external folder, or file them in a separate internal folder (then its link).
3. **Resolve each folder link** to a fixed folder ID: `CallGraph` GET `/shares/{encoded-link}/driveItem`, where the encoded link is `u!` followed by the base64url form of the link (no trailing `=`). Save `driveId`, the item `id`, and the folder `name`, and read the folder name back to the user so they can see it's the right one. If a link will not resolve, stop and say so; a profile with an unresolved folder is never saved.
4. **Show the complete profile and get a final confirm**, then save it to `clients/<client-slug>.md` (format below).
5. **Make sure the batch task exists** (see *Scheduling*). The first time any client is set up, also confirm the batch settings and save `clients/_settings.md`.
6. **Dry run this client** for the past 7 days and show the result: each meeting found, its classification, and the exact file names and folders it would produce. Nothing is written.
7. **Offer a one-time backfill**: an on-demand run for this client over a longer window the user chooses (for example the past 30 days). Run it only on a yes.
8. **Retire the old task.** If step 1 found an older per-client task for this client, tell the user to delete it in Cowork now. Left in place, it files and emails a second time.

Setup is complete when the profile is saved, the batch task exists exactly once, and the user has seen the dry run.

## Client Profiles

One file per client in this skill's `clients/` folder. The folder **ships empty** and is each user's own: a teammate who installs the skill starts blank and runs their own setup. Any file starting with `_` is settings, not a client.

```markdown
---
client: Contoso Manufacturing
keywords:                  # whole-word, case-insensitive match on the meeting title
  - Contoso
  - CMFG
client_domains:            # any attendee on one of these makes a meeting EXTERNAL
  - contoso.com
  - contosoadvisors.com
firm_domain: eidebailly.com
external_folder:
  link: https://tenant.sharepoint.com/:f:/r/sites/Contoso/Shared%20Documents/External%20Meeting%20Recordings
  drive_id: b!AbCdEf...
  item_id: 01ABCDEF...
  name: External Meeting Recordings
internal_handling: separate   # skip | same | separate
internal_folder:              # only when internal_handling is separate
  link: https://tenant.sharepoint.com/:f:/r/sites/Contoso/Shared%20Documents/Internal%20Recordings
  drive_id: b!AbCdEf...
  item_id: 01GHIJKL...
  name: Internal Recordings
---
Optional notes about this engagement.
```

`clients/_settings.md` holds the batch-wide settings:

```markdown
---
timezone: America/Chicago     # used for the scan window and every file name
notify: attention             # every_run | attention | errors_only
---
```

- **every_run**: email after every run, even when nothing was found.
- **attention**: email when anything was filed, or anything is on the *needs you* list.
- **errors_only**: email only when a run hits an error.

Every setting changes only when the user asks; edit the profile and confirm the change back.

---

## Batch Run

The same steps serve an on-demand run and a dry run: an on-demand run may narrow to one client or widen the window; a dry run stops before step 6 and reports what it would do.

1. **Load the settings and every profile.** A profile missing a folder ID is an error for that client, reported in the email; the other clients still run.
2. **Pull the calendar once.** One `ListCalendarView` call for the past **7 days** in the settings time zone, keeping online meetings that have ended and are not cancelled. Select subject, start, end, organizer, isOrganizer, attendees, and onlineMeeting. The duplicate check (step 6) makes re-scanning earlier days harmless, and a transcript still processing at run time is picked up by the next run.
3. **Match meetings to clients.** A meeting belongs to a client when a keyword appears in its title as a **whole word**, ignoring case. Then sort out the exceptions, each of which goes on the *needs you* list with its reason:
   - **Matches more than one client** → filed under neither.
   - **The user is not the organizer** (`isOrganizer` is false) → "you don't own this meeting; request access from the organizer".
   - **Channel meeting** (the join link contains `@thread.tacv2`) → "channel meeting; its files live in the team's SharePoint site".
4. **Classify each meeting, in this order, first match wins:**
   1. The title contains the word "internal" → **INTERNAL**.
   2. Any attendee's email is on one of the client's `client_domains` → **EXTERNAL**.
   3. Every attendee is on the `firm_domain` → **INTERNAL**.
   4. Anything else → **couldn't classify**, filed nowhere, listed on the *needs you* list with the attendee domains that didn't match.

   INTERNAL goes to the internal folder, or the external folder when `internal_handling` is `same`, or nowhere when it is `skip`. EXTERNAL goes to the external folder.
5. **Gather the files for each meeting.** A meeting can have several sessions (transcription stopped and restarted), and each session is filed separately.
   - **Transcripts:** `ListMeetingTranscripts` with the meeting's `onlineMeeting.joinUrl`, then `GetMeetingTranscript` in VTT format for each one. Each becomes a `.vtt` file. This is the only way to file a transcript: the Recordings `.mp4` of a transcript-only session opens empty once copied, because its transcript stays linked to the original.
   - **Recordings:** look in the **top level** of the user's OneDrive `Recordings` folder (ignore its subfolders) for `.mp4` files whose title matches the meeting (collapse repeated spaces before comparing) and whose timestamp falls within the meeting. Copy only files of **10 MB or more**. Smaller files are transcript-only shells with no video, whatever their name says; their transcript is already covered above.
   - Nothing found for a meeting is normal (not every meeting is transcribed). Count it; it is not an error.
6. **Name, check, and write each file.**
   - **Name:** `YYYY-MM-DD HHMM - Client Name - Meeting Title.ext` (see *File Naming*).
   - **Duplicate check:** list each destination folder once per run (`GetDriveChildren`) and skip any file whose name is already there.
   - **Transcript:** `CallGraph` PUT `/drives/{drive_id}/items/{item_id}:/{url-encoded file name}:/content?@microsoft.graph.conflictBehavior=fail`, `Content-Type: text/vtt`, body = the VTT text. A successful response returns the new file, which confirms it.
   - **Recording:** `CallGraph` POST `/drives/{source-drive}/items/{source-item}/copy?@microsoft.graph.conflictBehavior=fail` with the destination `parentReference` and the new `name`. The copy is asynchronous: a "202 Accepted" means started, not done. Confirm the file is in the destination folder before listing it as filed; if it isn't there yet, list it as "copy started, not yet confirmed".
   - Any failed call is an **error** for that meeting, with the message returned. The run moves on to the next meeting.
7. **Email the summary** per the `notify` setting, with `SendEmailWithAttachments` to the user. **An error always sends the email**, whatever the setting. One section per client listing each file filed and where, then one *needs you* list and one errors list. Every count in the email is the count of the itemized lines beneath it.

## File Naming

Every filed file gets one standard name. The raw Teams file name is never used.

`YYYY-MM-DD HHMM - Client Name - Meeting Title.ext`

| Segment | Value |
|---|---|
| `YYYY-MM-DD HHMM` | When that session started, in the settings time zone. For a transcript, its `createdDateTime`. For a recording, the timestamp in its file name: convert from UTC when it ends in `UTC`, and read it as local time when it has no suffix. |
| `Client Name` | The profile's `client`. |
| `Meeting Title` | The meeting subject. When it already starts with the client name, drop that repeat. Replace `/ \ : * ? " < > \|` with a hyphen, collapse repeated spaces, and trim. |
| `.ext` | `.vtt` for a transcript, `.mp4` for a recording. |

Examples:

- `2026-07-17 1400 - Contoso Manufacturing - Weekly Sync.vtt`
- `2026-07-17 1400 - Contoso Manufacturing - Weekly Sync.mp4`
- `2026-07-17 1435 - Contoso Manufacturing - Weekly Sync.vtt` (a second session of the same meeting)

**Changing a naming input re-files the past.** The duplicate check compares names, so renaming a client makes every earlier file stop matching, and the next run files those meetings again under the new name. Warn the user before saving such a change. Cleanup of the old files in SharePoint is manual.

## Scheduling

**One task for every client.** Create it with `SetupScheduledPrompt` (`execution_mode: separate`) the first time any client is set up. Confirm the cadence, time, and time zone at that moment:

- **Every weekday** (the default: Monday to Friday, 7:00 PM) → `frequency: Week`, `weekDays` Monday to Friday.
- **Weekly** on a chosen day → `frequency: Week`, `interval: 1`, `weekDays: [day]`.
- **Daily** → `frequency: Day`, `interval: 1`.

The task description is this one fixed instruction, naming no client: *"Run the client-recording-uploads skill's Batch Run for every client with a saved profile in its clients/ folder."* Clients are read from the folder, so adding or changing a client never touches the task.

Before creating it, call `GetScheduledPrompts` and confirm it doesn't already exist; two batch tasks file and email twice. To change or pause it, `GetScheduledPrompts`, then `EditScheduledPrompt`. The scheduler runs on holidays too; on a day with no meetings the run finds nothing.

## Tools

- **Calendar:** `ListCalendarView`
- **Transcripts:** `ListMeetingTranscripts`, `GetMeetingTranscript`
- **Recordings and destination folders:** `SearchDrive`, `GetDriveChildren`
- **Resolve links, copy, upload:** `CallGraph`
- **Summary email:** `SendEmailWithAttachments` (email only; a Teams message to yourself raises no notification)
- **Schedule:** `GetScheduledPrompts`, `SetupScheduledPrompt`, `EditScheduledPrompt`

## Output Examples

After setup:

```text
Saved: Contoso Manufacturing
• Matched by: "Contoso" or "CMFG" as a whole word in the title
• External (any @contoso.com or @contosoadvisors.com attendee) → External Meeting Recordings
• Internal ("internal" in title, or only @eidebailly.com attendees) → Internal Recordings
• Batch run: every weekday at 7:00 PM Central, covering 3 clients
• Dry run, past 7 days: 2 transcripts and 1 recording would be filed (listed below)
```

The run's email:

```text
Recording uploads: Fri, Jul 17
Contoso Manufacturing: 3 files filed
  • 2026-07-17 1400 - Contoso Manufacturing - Weekly Sync.vtt → External Meeting Recordings
  • 2026-07-17 1400 - Contoso Manufacturing - Weekly Sync.mp4 → External Meeting Recordings
  • 2026-07-16 0930 - Contoso Manufacturing - Internal Prep.vtt → Internal Recordings
Fabrikam: nothing new
Needs you: 1
  • "Contoso Exec Review" (Jul 16): you don't own this meeting; request access from the organizer
Errors: none
```

## Guardrails

- **Confirm, don't infer.** Every profile field is confirmed with the user and every folder link is resolved at setup. Folder links, domains, and client names come only from the user.
- **Copy and upload only.** SharePoint files can be added but not deleted with these tools, so fixing a misfiled file is manual. That is why an unclear meeting goes on the *needs you* list instead of into a folder.
- **Client isolation.** A meeting is filed under exactly one client, and one client's error never stops another client's run.
- **Silence means nothing happened.** Errors always email, so a quiet inbox never hides a broken run.
- **Portable.** Settings live in `clients/`, and the skill itself holds no personal or client data, so it can be shared across the team.
