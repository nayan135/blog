---
title: 'How I built a fully automated blog writing pipeline with n8n and local LLMs'
description: 'A look into how I built a hands-off blog drafting pipeline using n8n, Ollama, and Git to publish content without manually writing Markdown.'
pubDate: '2026-07-06'
---
I get tired of the blank page. I have dozens of half-baked ideas sitting in my Obsidian vault that never turn into actual posts because turning a raw bulleted list into a structured Markdown file takes more activation energy than I usually have after a workday in Kathmandu.

Two weeks ago, I decided to automate the boring parts. I wanted a system where I could drop a quick, messy voice transcript or a chaotic list of bullet points into a folder, and have a self-hosted pipeline output a formatted, clean Markdown draft ready for my final edit.

I built this using n8n, Ollama running llama3.1 locally on my home server, and a simple Git trigger.

### The architecture of the pipeline

The whole system runs self-hosted. I don't like paying OpenAI token fees for messy drafts that I am going to heavily edit anyway. 

Here is how the data flows:
1. I save a raw text file (`idea-name.txt`) into a specific input folder in my Obsidian vault.
2. Syncthing syncs this folder to my home server.
3. A local cron job or n8n Local File Trigger detects the new file.
4. n8n reads the file and sends it to Ollama with a highly specific system prompt.
5. Ollama generates the post body and YAML frontmatter.
6. n8n writes the output directly to my local Hugo content directory as `idea-name.md`.
7. n8n triggers a local bash script to run a git commit and push to my private staging branch.

### The n8n workflow setup

Inside n8n, the setup is surprisingly light. You do not need complex agent nodes for this. I used standard HTTP Request nodes and the local execution tool.

The first block is a **Local File Trigger** node. It watches `/home/nayan/vault/ideas/*.txt` for new files. When a file appears, it passes the raw content to the next node.

The heavy lifting happens in the **Ollama node** (or a standard HTTP node pointing to `http://localhost:11434/api/generate`). 

Here is the exact prompt template I write into the node:

```text
System: You are an assistant formatting technical notes into a clear Markdown draft. Keep the tone conversational, direct, and technical. Do not use grand intro statements or concluding summaries. Output only the frontmatter and the body. Do not explain your choices.

User input:
{{ $json.content }}
```

I noticed that if I do not explicitly tell Ollama to shut up and just write the markdown, it starts the file with "Here is your blog post:" which breaks the Hugo build immediately.

### Handling the Git push

Once Ollama returns the generated text, the next node is an **Execute Command** node. I do not use n8n's built-in Git integrations because they feel clunky for simple file writes. Instead, I write the output using a simple shell command node inside n8n:

```bash
cat << 'EOF' > /home/nayan/projects/personal-site/content/posts/{{ $json.filename }}.md
{{ $json.generated_text }}
EOF
```

After writing the file, another Execute Command node runs a bash script that handles the Git staging environment:

```bash
cd /home/nayan/projects/personal-site
git checkout draft-pipeline
git add content/posts/
git commit -m "auto: draft generated for {{ $json.filename }}"
git push origin draft-pipeline
```

This pushes the draft to a specific staging branch on GitHub. I have a Vercel preview deployment linked to this branch. Within two minutes of dropping a text file into my Obsidian vault, I get a live URL where I can read the drafted post on my phone.

### Where this approach breaks (and how to fix it)

Local models are stupid sometimes. Llama3.1 8B is fast on my RTX 3060, but it occasionally hallucinates frontmatter fields or writes JSON instead of YAML. 

To prevent my Hugo build from failing because of a malformed date or missing quotation mark in the frontmatter, I added a JS Code node in n8n right after the Ollama step. This node uses a basic regex to validate that the output starts with `---` and ends with `---` before writing the file. If the validation fails, n8n sends me a Telegram notification with the error, and I know I need to tweek my raw notes or rerun the node.