---
title: 'How I built a fully automated blog workflow with n8n and Gemini'
description: 'A look behind the scenes of my self-hosted n8n setup that pulls ideas from Todoist, drafts posts using Gemini, and commits them directly to GitHub.'
pubDate: '2026-07-06'
---
I write a lot of code, but writing blog posts consistently is hard. A few weeks ago, I decided to automate the boring parts of my writing process. I did not want an AI to write my entire blog without my input. That results in dry, generic garbage. Instead, I wanted an automated partner that takes my messy, half-baked technical notes and turns them into structured Markdown drafts that I can edit and publish with one click.

Here is how I built a hands-off pipeline using a self-hosted n8n instance, Todoist, the Gemini API, and GitHub.

### The architecture of the pipeline

The whole system runs on a Docker-compose setup on my tiny VPS here in Kathmandu. The workflow has four main stages.

First, the trigger. I use Todoist to manage my life. When I get an idea for a post, I create a task in a specific "Blog Ideas" project. I write a quick title and dump my messy thoughts, code snippets, or console logs into the task description. n8n polls Todoist every hour looking for new tasks in this project that have a specific label like `#ready-to-draft`.

Second, the data extraction. Once n8n finds a task, it pulls the title and the raw description. It also fetches any comments I might have added to that task, which usually contain extra terminal outputs or links to GitHub issues I found helpful.

Third, the prompt engineering. This is where most of my time went. I pass this raw dump to the Gemini 1.5 Pro model via the Google Gemini node in n8n. I do not use a simple "write a blog post about this" prompt. My system prompt is highly specific. It instructs the model to use my personal tone, maintain a direct developer-to-developer style, and format the output as valid Markdown with specific frontmatter. I also explicitly forbid the AI from using generic corporate buzzwords. 

Fourth, the commit. The output from Gemini goes straight to a GitHub helper node. It creates a new branch in my website repository and commits a new `.md` file inside my `src/content/blog/` directory. The file name is automatically generated from the slugified task title. Finally, n8n leaves a comment on the original Todoist task with a link to the GitHub branch and marks the task as complete.

### Handling the API costs and rate limits

I chose Gemini 1.5 Pro because the free tier limits are quite generous for a personal blog. I do not post ten times a day, so I stay well within the requests-per-minute boundary. If you try to run this with OpenAI, you will need to load a pre-paid balance, which is fine, but Gemini is incredibly easy to set up with a simple API key from Google AI Studio.

One issue I ran into early on was JSON parsing errors. Sometimes the LLM would output the Markdown with triple backticks wrapping the frontmatter, which broke my Hugo build process. I fixed this by using an n8n Code node running a short JavaScript regex. It strips away any accidental markdown code blocks that the LLM wraps around the actual code output before committing to GitHub.

Here is the regex I used in the Code node to clean up the output:

```javascript
let content = $input.item.json.text;
// Remove leading markdown code blocks if the model wrapped the response
content = content.replace(/^```markdown\s*/i, '');
content = content.replace(/^```html\s*/i, '');
content = content.replace(/\s*```$/, '');
return { cleanContent: content };
```

### The final editorial human check

This setup does not publish directly to production. That would be risky. Instead, the commit lands on a separate branch. My site is hosted on Vercel, which automatically generates a preview deployment for every new branch. 

When n8n finishes the workflow, I get a push notification on my phone from the GitHub app. I can click the preview link, read the draft on my phone while drinking tea, and see how it looks. If I like it, I merge the pull request on GitHub, and the site rebuilds. If it needs tweaks, I open the file in my browser on GitHub, edit the text directly, and merge it.

This system keeps me in control while removing the friction of creating the initial file, writing the frontmatter metadata, and setting up the structure. I just dump my raw thoughts into my todo list, and a few minutes later, I have a real draft waiting for me.