---
title: 'The n8n binary data bug that only showed up after 4 hours'
description: 'A war story about chasing an ENOENT error in n8n''s binary data manager, caused by execution pruning deleting files mid-workflow.'
pubDate: '2026-07-04'
---
Last month I got a Slack message at 11pm from a client: "the invoice bot is broken again." Again, because this was the third time in two weeks, and every time I looked at it in the morning everything worked fine. That's the worst kind of bug — the one that fixes itself before you get to it.

The workflow itself wasn't complicated. A webhook receives an email attachment (usually a PDF invoice), n8n runs it through an OCR node, extracts totals and vendor names, then hits a Wait node so someone on the finance team can approve it via a link before it gets pushed to their accounting system. Nothing fancy. It had been running for about six weeks without issues before this started.

The error, when I finally caught it in the logs, was this:

`ENOENT: no such file or directory, open '/home/node/.n8n/binaryData/15791023-ab.bin'`

Thrown from inside n8n's `FileSystemManager`, right when the workflow resumed after someone clicked the approval link. The workflow would restart from the Wait node, try to grab the binary data it had stored before pausing, and just... not find the file. The file that should have had the invoice PDF in it.

First instinct: disk issue. I checked the volume mount on the Docker container, checked `df -h`, plenty of space, no weird permission errors, nothing in dmesg. I SSH'd into the box and just watched the binaryData folder with `watch -n 1 ls`. And that's when I saw it — files were disappearing on their own, roughly every hour, in batches.

That's when it clicked. n8n has a setting called `EXECUTIONS_DATA_PRUNE` that defaults to true, with `EXECUTIONS_DATA_PRUNE_MAX_AGE` set to 336 hours by default (14 days) in some versions, but this client's instance had it overridden to 4 hours, set by whoever configured the server originally, presumably to keep the execution history table small. The pruning job doesn't just delete rows from the execution log. It also cleans up any binary data files tied to those executions, because n8n considers a workflow "done" once it hits a Wait node in some versions depending on how the wait is configured, or the cleanup job just doesn't know the difference between a finished execution and one that's sitting paused waiting for a human to click a link.

So if the approver didn't click within 4 hours, the pruning cron would sweep through, see an execution that looked stale, and delete the .bin file the PDF was stored in. The workflow itself stayed alive, waiting patiently for an approval that, when it finally came, pointed to a file that no longer existed.

The annoying part is how rarely this actually triggered. Most invoices got approved within an hour or two, so this only bit us when someone was slow to check their email, which is exactly the kind of intermittent trigger that makes a bug take three weeks to diagnose instead of three minutes.

The real fix ended up being two changes. First, I bumped `EXECUTIONS_DATA_PRUNE_MAX_AGE` up to 72 hours, which is a band-aid but a reasonable one for this use case. Second, and more importantly, I restructured the workflow so it doesn't hold onto the raw binary across the Wait node at all. Instead of storing the PDF and waiting, it now stores just the metadata (message ID, attachment ID) and re-fetches the attachment from the email provider's API after approval. Binary data lives for the seconds it's actually being processed, not for however long it takes someone to check their inbox.

If you're running n8n self-hosted and using Wait nodes for anything that involves binary data, go check your `EXECUTIONS_DATA_PRUNE_MAX_AGE` right now. The default assumption baked into that pruning logic is that once an execution is old, nothing about it matters anymore. That's not true if a human is sitting in the middle of your workflow, and humans are slow, unpredictable, and occasionally on vacation.

The lesson I keep relearning with n8n: binary data isn't a first-class citizen in the way you'd want it to be once a workflow spans more than a few seconds. Treat it as disposable, refetch it when you need it again, and don't trust it to survive anything that involves waiting on a person.