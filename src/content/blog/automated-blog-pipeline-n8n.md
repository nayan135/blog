---
title: 'How I built a fully automated blog writing pipeline with n8n'
description: 'A look inside my self-hosted n8n workflow that generates draft blog posts from raw markdown notes using Claude.'
pubDate: '2026-07-04'
---
I have dozens of half-baked ideas sitting in my local Obsidian vault. They are mostly messy scribbles, terminal outputs, and random code snippets that I paste while trying to fix a bug at 2 AM. Turning these raw notes into a coherent draft takes more activation energy than I usually have after a long day of client work here in Kathmandu.

So I decided to automate the boring parts. I built an asynchronous pipeline using n8n and the Anthropic API that takes a rough markdown file from a specific folder, processes it, structures it, and dumps a formatted markdown draft straight into my website's content directory.

Here is how the setup works, why I chose n8n, and the exact prompt logic I use to avoid making my blog sound like a generic corporate newsletter.

### Why n8n instead of a custom script?

I initially started writing a Node.js script with `fs-extra` and the `@google/generative-ai` SDK to handle this. But managing state, retries, and file system triggers in a plain script gets messy quickly. 

I run a self-hosted instance of n8n on a cheap Hetzner VPS. It has a native Local File Trigger node. This node watches a specific directory on my disk. The moment I save a file ending in `.todo.md` in that directory, the workflow kicks off. If the API rate limits me or the network drops, n8n handles the retries automatically. I do not have to write boilerplate error-handling code.

### The workflow architecture

The pipeline consists of five main nodes:

1. **Local File Trigger**: Monitors `/data/obsidian/drafts` for any new files matching `*.todo.md`.
2. **Read Binary File**: Loads the actual content of the markdown file into the n8n execution context.
3. **Extract Metadata**: A simple JavaScript code node that splits the YAML frontmatter from the actual messy notes.
4. **Anthropic Chat Node**: Passes the raw notes to Claude 3.5 Sonnet with a highly restrictive system prompt.
5. **Write Binary File**: Saves the output as a clean `.md` file directly into my Astro project's `src/content/blog/` directory, then deletes the original `.todo.md` file to prevent infinite loops.

### The prompt that prevents generic AI garbage

If you ask an LLM to "write a blog post," you get a wall of text filled with words like *delve, testament, foster, and landscape*. It sounds incredibly fake. To combat this, my n8n OpenAI/Anthropic node uses a system prompt that actively bans these linguistic patterns.

Here is the exact instruction set I pass to the model:

```text
You are an experienced systems developer. You are converting rough notes into a technical blog post.

Follow these strict rules:
- Write in a direct, first-person, casual tone.
- Do not use words like: delve, pivotal, testament, seamless, robust, leverage, or key takeaways.
- Avoid starting paragraphs with transitions like "First and foremost," "In conclusion," or "Furthermore."
- Keep code snippets exactly as they are in the source notes. Do not optimize them unless specifically asked.
- If the source notes contain a specific error message, keep that error message intact.
```

By restricting the vocabulary, the model has to write like a normal person. It cannot fall back on its default corporate-safe training data patterns.

### Handling the file system sync

Since my n8n instance runs inside a Docker container, it cannot access my host machine's files by default. I had to mount my local Obsidian vault directory and my website's project folder as Docker volumes in my `docker-compose.yml` file:

```yaml
volumes:
  - /home/nayan/vaults/personal:/data/obsidian
  - /home/nayan/projects/nayan135.xyz:/data/website
```

This gives n8n direct read and write access. When I save `tailscale-routing.todo.md` in Obsidian, n8n grabs it, runs the LLM chain, and outputs `tailscale-routing.md` into my website's blog directory. 

### The human in the loop

I do not auto-publish these files. That would be a disaster. The workflow places the generated files into my local Git working directory as untracked files. 

When I run `git status`, I see the new draft. I open it in VS Code, read through it, fix any weird assumptions the LLM made, and format the code blocks. Once I am happy with it, I run a git commit and push it to production. 

This setup saves me about an hour of formatting and structuring per post. I do the rough thinking, n8n does the heavy lifting of organizing the raw output, and I do the final editorial pass.