---
title: 'How I built a fully automated blog writing workflow with n8n and AI'
description: 'A look into how I automated my content pipeline using self-hosted n8n, Claude, and GitHub APIs without losing my personal voice.'
pubDate: '2026-07-04'
---
I hate writing draft outlines. I love having ideas, and I love editing final drafts, but the middle part where you stare at a blank VS Code window trying to structure a technical post is miserable. Last month, I decided to automate that friction away. 

I built a self-hosted n8n workflow that takes a raw, half-baked thought from my phone, refines it with Claude, structures it into markdown, and opens a pull request directly on my GitHub repository for this site. Here is exactly how it works, what broke during setup, and how you can spin up something similar.

### The architecture

I run n8n on a cheap Hetzner VPS using Docker Compose. The pipeline relies on four distinct stages:

1. **The Trigger**: A webhook URL that I trigger using a simple iOS Shortcut. When I have an idea while walking around Kathmandu, I open the shortcut, type a quick brain dump, and send it.
2. **The Refiner**: An n8n basic LLM chain node connected to Anthropic's Claude 3.5 Sonnet. 
3. **The Parser**: A JavaScript code node that extracts the JSON payload from the AI response and formats the frontmatter.
4. **The Publisher**: A couple of HTTP Request nodes that talk to the GitHub API to create a new branch, commit the markdown file, and open a PR.

### Writing the system prompt

The hardest part of this setup was getting Claude to stop sounding like a marketing department. If you ask an LLM to "write a blog post," it defaults to corporate corporate-speak filled with words like "revolutionize" and "delve." 

I had to write a highly restrictive system prompt. I explicitly forbid it from using passive voice, exclamation marks, or any introductory fluff. I also feed it three of my previous blog posts as few-shot examples so it can match my rhythm and sentence structure. 

Here is a snippet of the instruction prompt I settled on:

```text
Write in the first person. Start directly with the technical problem. 
Do not write an introduction paragraph explaining what the technology is. 
Keep sentences short. Use contractions. If the input draft contains a code snippet, 
keep that snippet exactly as is. Output only valid JSON with keys 'title', 'slug', and 'content'.
```

### Handling the GitHub API dance

Most people stop their automation at "send an email with the text." I wanted a working git branch. Doing this through the GitHub API without installing a heavy local git client inside the n8n container requires a specific sequence of API calls.

First, you need to get the SHA of the main branch's latest commit. You do this with a GET request to `/repos/{owner}/{repo}/git/ref/heads/main`.

Second, you create a new branch by sending a POST request to `/repos/{owner}/{repo}/git/refs` with the new branch name and the SHA you just fetched.

Third, you write the file. The GitHub API expects the file content to be Base64 encoded. If you try to send raw markdown, the API returns a 422 Unprocessable Entity error. I learned this the hard way after debugging bad requests for an hour. In n8n, you can use a basic Code node to encode your payload:

```javascript
const content = $input.item.json.content;
const base64Content = Buffer.from(content).toString('base64');
return { base64Content };
```

Then you send a PUT request to `/repos/{owner}/{repo}/contents/{path}` with the base64 string, the branch name, and a commit message.

Finally, you hit `/repos/{owner}/{repo}/pulls` to open the PR. 

### What went wrong

The biggest headache was rate limits and timeout issues. Sometimes Anthropic takes 15 seconds to generate a full post. By default, n8n webhook nodes might timeout if the downstream nodes take too long to resolve. 

To fix this, I changed the Webhook node settings in n8n. I set the "Response Mode" to "Immediately" with a 200 OK status. This lets my phone know the message was received instantly. The actual processing runs in the background. If something fails mid-way, n8n sends me a Discord notification using a simple Discord webhook node attached to the Error Trigger.

Now, when I sit down at my desk, I don't start with a blank page. I open GitHub, look at my pending pull requests, checkout the generated branch, and start editing. It saves me about two hours of procrastination per post.