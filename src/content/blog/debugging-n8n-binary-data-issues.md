---
title: 'How I broke and fixed n8n\'s binary data handling in production'
description: 'A real-world debugging story of why n8n workflows crash when handling large PDF files on low-memory VPS instances, and how to fix it.'
pubDate: '2026-07-03'
---
A client called me last Tuesday because their automated invoicing pipeline was failing silently. They use n8n self-hosted on a tiny 2GB RAM Hetzner VPS to pull PDF invoices from an IMAP email node, compress them, and dump them into an S3 bucket. 

Every morning at 9:00 AM, the n8n container would just die. No clean exit codes, no helpful error logs in the UI, just a sudden `docker-compose` restart. 

I looked at the system logs first. Running `dmesg -T | grep -i -E 'oom-kill|out of memory'` confirmed my suspicion. The Linux kernel was killing the Node.js process running n8n because it ran out of RAM. 

Here is how n8n handles binary data by default. When n8n downloads an email attachment, it keeps that file in memory as a base64 encoded string. If a supplier sends a 15MB PDF invoice, Node.js needs to allocate that 15MB in heap memory. But because of how base64 encoding works, that 15MB file actually takes up about 20MB of RAM. 

Then, the workflow passes that binary data to the next node. In older versions of n8n, or if you have not configured your environment variables correctly, n8n duplicates this data in memory for every single node execution step. If your workflow has five steps to process that PDF, you are suddenly looking at over 100MB of RAM for a single execution. Multiply that by ten concurrent emails, and your 2GB VPS is toast.

To fix this, we have to force n8n to stop using RAM for binary data. 

n8n has an option to write binary data straight to disk instead of keeping it in memory. It retains a reference pointer in memory, but the actual heavy lifting happens on your SSD. 

I opened our `.env` file for the docker-compose setup and added this crucial line:

```env
N8N_DEFAULT_BINARY_DATA_MODE=filesystem
```

By default, this is set to `memory`. Changing it to `filesystem` tells n8n to write these files to a temporary directory inside the container. 

But just setting that variable is not enough. If your container restarts, or if you do not clean up those files, your disk will fill up fast. You need to configure the execution data pruning settings too. I added these lines to the environment file:

```env
EXECUTIONS_DATA_PRUNE=true
EXECUTIONS_DATA_MAX_AGE=168
EXECUTIONS_DATA_PRUNE_TIMEOUT=3600
```

This setup tells n8n to delete execution data older than 168 hours (7 days) and run the cleanup job every hour.

After saving the `.env` file, I ran `docker compose up -d` to recreate the container. I triggered the workflow manually with a batch of twenty test emails containing 10MB PDF files. 

I watched the memory usage using `htop`. Instead of spiking to 1.8GB and invoking the OOM killer, the Node.js process hovered around a stable 320MB. You could see the temp files being created in `/home/node/.n8n/binaryData` inside the container and then disappearing after the workflow finished.

If you are running n8n on a budget VPS, do not leave your binary data mode on `memory`. Switch it to `filesystem` before you get a frantic call from a client whose automated workflows have been dead for three days.