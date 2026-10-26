# Client profiles

One file per client, created the first time you set up **client-recording-uploads** for that client. Each file stores the client's meeting keywords, email domains, and the SharePoint folders its recordings and transcripts go to. `_settings.md` holds the batch-wide time zone and email setting.

This folder ships **empty** on purpose. It is the skill's settings store, not shared data: your profiles stay in your own copy, and a teammate who installs this skill starts with a blank folder and runs their own setup.

There is **one** scheduled run that covers every client with a profile here. Adding a client means adding a profile, not another scheduled task.

You don't need to edit these by hand. To change a client's settings, say "reconfigure recording uploads for [client]".
