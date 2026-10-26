# client-recording-uploads

A Microsoft Copilot Cowork skill that files your clients' Teams meeting transcripts and recordings into each client's SharePoint folders, on a schedule, without you lifting a finger. It sorts each meeting into internal or external, saves the transcript as a text file, copies any video recording, and emails you a summary. It never deletes anything.

## What you'll need

- Access to Microsoft Copilot Cowork.
- For each client you want to set up:
  - **A word or two from the meeting titles**, usually the client's name or an abbreviation (for example `Contoso` or `CMFG`). This is how the skill spots that client's meetings on your calendar.
  - **The client's email domain**, the part of their email address after the @ (for example `contoso.com`).
  - **A link to the SharePoint folder** where that client's recordings should go. To get it, open the folder in SharePoint and use **Copy link**. If you want internal meetings in their own folder, have a second link ready.

## Install

1. Download this repo as a zip (Code → Download ZIP on GitHub), or clone it.

   ![The Code button and Download ZIP option on a GitHub repo page](docs/download-zip.png)

2. In Copilot Cowork, go to **Customize → Skills**, click the arrow on the **Add** button, and choose **Upload skill**.

   ![Upload skill button in Copilot Cowork's Customize > Skills screen](docs/upload-skill.png)

3. Upload the zip from step 1.
4. Run it for a client for the first time by saying "set up recording uploads for [client]". It will walk you through one-time setup and save your own profile under `clients/`.

Your client profiles never leave your own copy: the `clients/` folder ships empty and is gitignored, so nothing you configure gets shared back here.

## What setup looks like

Setup is a short conversation in Cowork. It asks a few questions at a time:

1. The client's name, as you want it to appear in file names.
2. The keywords that mark a meeting as this client's.
3. The client's email domains. Add any outside advisor who regularly joins this client's meetings.
4. What to do with internal meetings: skip them, put them in the same folder, or put them in a separate folder.
5. The SharePoint folder links. It reads each folder's name back to you so you can confirm it's the right one.

The first time you set up any client, it also asks your time zone, how often to run (the default is every weekday at 7:00 PM), and when you want the summary email.

Before anything is copied, it shows you a **dry run**: a list of the meetings from the past 7 days and exactly what it would file where. Then it offers to file older meetings once, for example the past 30 days. After that, it runs on its own.

## What happens after that

- **One scheduled run covers all your clients.** Adding a client adds it to the same run, not a new one.
- **Each run looks back 7 days** and skips anything already filed, so nothing gets filed twice.
- **Files get one standard name**, for example `2026-07-17 1400 - Contoso Manufacturing - Weekly Sync.vtt`. The `.vtt` file is the transcript, a plain text file you can open in Notepad or any text editor. The `.mp4` file is the video, when the meeting was recorded.
- **You get an email** listing what was filed and where. It also has a **Needs you** list of meetings it didn't file because it couldn't do so safely, each with the reason. Those are yours to handle by hand.
- **If something goes wrong, you always get an email**, whatever your email setting. A quiet inbox never hides a broken run.

## Good to know

- **It only files meetings you organized.** For a meeting someone else set up, it lists the meeting on your Needs you list so you can ask the organizer for the recording.
- **Only transcribed or recorded meetings have anything to file.** A meeting with neither is skipped quietly.
- **Meetings held in a Teams channel are left alone.** Their files already live in that team's SharePoint site.
- **It can add files to SharePoint but never moves or deletes them.** If something lands in the wrong folder, move it yourself.
- **Renaming a client files its last 7 days of meetings again** under the new name. Older files keep the old name.
- **If you delete the scheduled run in Cowork yourself, tell Cowork** ("I deleted the recording uploads schedule"). The skill keeps a note that the run exists, and clearing it lets the next client setup create a new one.

## Things you can say

| Say this in Cowork | What happens |
|---|---|
| "set up recording uploads for [client]" | Adds a new client |
| "reconfigure recording uploads for [client]" | Changes a client's settings |
| "dry run recording uploads" | Shows what would be filed, without copying anything |
| "run recording uploads now" | Runs right away for every client. Add a client name, or "for the past 30 days", to narrow or widen it |

## What's in this repo

- `SKILL.md`: the skill's instructions
- `clients/`: where your own client profiles get saved (empty here on purpose)
- `docs/`: the screenshots used in this page

## Updates

This repo is maintained by [Daniel Daou](mailto:ddaou@eidebailly.com). Pull the latest whenever there's a new version. Your client profiles are not part of the zip, so when you upload a new version, check that your existing clients carry over. If they don't, run setup again for each one.
