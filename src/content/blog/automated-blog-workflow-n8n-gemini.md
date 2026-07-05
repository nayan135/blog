---
title: 'How I built a fully automated blog workflow with n8n and Gemini'
description: 'I got tired of copy-pasting outlines. Here is how I chained n8n, local markdown files, and Gemini to automate my draft generation.'
pubDate: '2026-07-05'
---
I write a lot of markdown files. Up until last week, my process for draft generation was a mess of open tabs. I would write a rough outline in VS Code, copy it, paste it into a browser tab running Google AI Studio, coax Gemini 1.5 Pro into expanding my bullet points, copy the response, paste it back into my editor, and then spend twenty minutes cleaning up weird formatting choices. 

It was tedious. I wanted a way to just commit a raw markdown outline to a specific folder in my local git repo and have a polished draft appear in my drafts folder five minutes later. 

I built exactly that using n8n and the Gemini API. Here is the exact setup, the hiccups I ran into, and how you can replicate it.

### The architecture of a lazy writer

The whole system runs on self-hosted n8n running in a Docker container on my home server. I don't use cloud triggers because I like keeping my local files local until they are ready for GitHub. 

Instead, I use a local directory watcher. The workflow triggers whenever a new file lands in `/Users/nayan/code/blog/raw_ideas/` with a `.md` extension.

Here is how the data flows once n8n picks up the file:
1. Read the raw markdown outline.
2. Clean up any weird system metadata (like hidden DS_Store files that macOS loves to create).
3. Send the raw outline to Gemini 1.5 Pro with a highly specific system prompt.
4. Receive the generated draft.
5. Write the draft to `/Users/nayan/code/blog/drafts/` with the original filename intact.
6. Move the raw outline file to an archive folder so it doesn't trigger the workflow again.

### Keeping the AI on a leash

The hardest part of this setup wasn't the n8n logic. It was stopping the LLM from writing corporate garbage. If you let an LLM write a blog post with default settings, it will choke the text with words like 'vibrant' and 'seamless'. To combat this, I had to get aggressive with the system instructions in my n8n Gemini node.

I set the temperature to 0.7. If you go too low (like 0.2), the output sounds like a dry manual. If you go too high (above 1.0), it hallucinates fake npm packages.

This is the exact system instruction block I ended up using in n8n:

```text
You are a technical writer who values brevity. 
Do not use passive voice. 
Do not start sentences with 'In today's digital age' or similar filler. 
Do not use bulleted lists where a simple paragraph works better. 
Write in the first person. 
Use contractions. 
Keep the technical details exact.
```

I also had to explicitly tell it to output raw markdown without wrapping the entire response in triple backticks. If you don't do this, your output file ends up starting with a literal ```markdown tag, which breaks my Hugo parser.

### Setting up the n8n directory watcher

To make this work locally, you need to mount your local blog directory to your n8n Docker container. If you run n8n on Docker, it can't see your host system files by default. 

My docker-compose volume mounts look like this:

```yaml
volumes:
  - /Users/nayan/code/blog:/data/blog
```

In n8n, inside the 'Local File Trigger' node, I set the path to `/data/blog/raw_ideas/` and set the event to 'File Created'. 

Next, I added a 'Read Binary File' node. The trigger node only tells n8n that a file exists. You need this second node to actually load the file content into memory. I set the file path in this node dynamically using an expression: `{{ $json.path }}`.

### Parsing the content and calling the API

Once the file is read, n8n treats it as binary data. I used a 'Code' node running basic JavaScript to convert that binary buffer into a UTF-8 string that the Gemini node can read. 

```javascript
const binaryData = items[0].binary.data;
const decodedText = Buffer.from(binaryData.data, 'base64').toString('utf-8');
return [{ json: { text: decodedText } }];
```

This text then goes straight into the Gemini node. I use the 'Generate Content' action with the `gemini-1.5-pro` model. It handles long contexts much better than the flash model, which is useful when my outlines contain long code snippets I want to preserve.

### Writing the output back

The final step uses the 'Write Binary File' node. I set the target path to `/data/blog/drafts/{{ $node["Local File Trigger"].json.name }}`. This ensures the output file keeps the exact name of my input outline.

Lastly, I use an 'Execute Command' node to run a simple shell command that moves the original file out of the `raw_ideas` directory: 

`mv "/data/blog/raw_ideas/{{ $node["Local File Trigger"].json.name }}" "/data/blog/archive/"`

Without this final step, the file trigger might loop or keep processing the same file if n8n restarts.

### The result

Now, when I get an idea, I open my terminal, type `touch raw_ideas/css-grid-tricks.md`, and quickly scribble down three bullet points and a code snippet. I save the file. 

By the time I make a cup of tea, a clean, 600-word draft is waiting for me in my editor. It still needs my personal touch, a few edits, and some formatting tweaks, but the blank-page anxiety is completely gone.