01: What Is an AI Agent?
Let me start where you already live

You've been writing automation for twenty years. Shell scripts, Terraform, Jenkins, Ansible. Every one of them works the same way:

if [ $(df -h /u01 | awk 'NR==2{print $5}' | tr -d '%') -gt 90 ]; then
    find /var/log -mtime +7 -delete
fi


You wrote every step. The machine follows exactly. It never surprises you.

And that predictability is precisely why we trust automation. It's a feature, not an accident.

Now here's the flaw you've lived with your whole career

A script only handles what you predicted.

You wrote that rule picturing log files filling the disk. But one night at 2am, /u01 fills because Oracle archive logs grew to 180 GB in /u01/oradata/archive.

Your script runs. Looks in /var/log. Finds nothing older than 7 days. Deletes nothing. Exits zero. Reports success.

The disk is still full. The script "worked." Nobody was helped.

This is why every runbook in every organisation you've ever worked in goes stale. A runbook is a photograph of what somebody once imagined might happen — taken on a day that has now passed. Reality keeps producing situations faster than humans can write them down.

You know this. You've lived it. Now let me show you what changes.

The agent has a different contract

An agent isn't given steps. It's given a goal:

"Find out why this server is unhealthy and tell me how to fix it."

Nobody told it to run df -h. It decided that.

Nobody told it to then run du -sh /u01/*. It decided that too — because of what df -h returned.

Nobody told it to query Oracle. It decided that because the directory it found was called oradata, and it reasoned that a database was involved.

Here's your one line. Commit it to memory:

A script follows instructions. An agent pursues a goal and works out the instructions itself.

Say it out loud once. Everything in the next sixteen lessons hangs off that sentence.

So what makes this possible?

The LLM. Large Language Model. The thing behind ChatGPT, Claude, Gemini.

Strip away every bit of hype and marketing, and it does exactly one narrow trick:

Given some text, it predicts sensible text that would follow.

That's it. That's the whole mechanism. Nothing more.

Give it this:

Alert: disk 97% on prod-db-01
Output of df -h:         /u01 is 97% full
Output of du -sh /u01/*: /u01/oradata/archive = 180GB


And it produces:

"Archive logs are consuming the volume. Purge old archives and add a datafile."

Why does that work? Because it has read a staggering volume of technical writing — documentation, Stack Overflow, Oracle manuals, forum threads, blog posts. It has encountered that shape of problem followed by that shape of answer thousands of times.

It does not understand your server. It recognises a pattern.

Hold that distinction. It'll matter enormously when we get to safety.

And now the limitation that is the entire reason your product exists

Sit up for this one.

An LLM cannot DO anything. It can only produce text.

It cannot SSH. It cannot query a database. It cannot read a file. It cannot call an API.

Test it yourself. Open ChatGPT right now and ask: "Why is my disk full on prod-db-01?"

You'll get a confident, fluent, well-structured, professional answer. And it will be a guess — because ChatGPT has never seen prod-db-01 and never will.

So how does an "agent" act at all?

Somebody writes ordinary code that does the acting.

In your product, that's:

backend/app/services/tool_registry.py    ← the dispatcher
backend/app/services/ssh_service.py      ← actually SSHes
backend/app/services/mcp_service.py      ← actually queries Oracle


The loop:

1. LLM produces text:    "run df -h on prod-db-01"
2. Your Python reads it
3. Your Python ACTUALLY SSHes in and runs it
4. Your Python sends the REAL output back
5. LLM produces text:    "now check /u01 subdirectories"
6. Repeat until: "I have enough. Here's the diagnosis."

The LLM is the brain. Your Python is the hands. The brain never touches anything directly.
Why this one architectural fact wins you deals

Because of that separation, you can walk into a bank's security review and say — truthfully:

"Our AI has no execution rights. It proposes actions. Our own code decides what actually runs, under our credentials, with our controls. There is no path from a model output directly to a shell."

That's genuinely true of your product. It's one of the strongest sentences you own.

We'll stress-test exactly how well it holds up in Lesson 10 — there's nuance — but the architecture is sound.

The three ingredients

Every AI agent on earth, including yours:

#	Ingredient	What it is	In your product1	A brain	The LLM	GPT-4o / Claude / Gemini — switchable from the UI
2	Hands	Code that can actually act	tool_registry.py, ssh_service.py, mcp_service.py
3	A goal + freedom	An objective, not a recipe	"Diagnose this alert"
Remove the hands → chatbot
Remove the goal and freedom → script
All three → agent

That's your diagnostic test. Someone shows you an "AI agent" at a conference? Ask which of the three it actually has. Most have two.

The table you'll use in front of customers
	Runbook	AI AgentHandles what nobody predicted	❌	✅
Goes out of date	Always	No
Must be found at 2am	❌ Yes	✅ No
Available instantly	If someone can find it	Always
Same quality every time	Depends who's reading	✅ Yes
Explains its reasoning	❌	✅

Your own Chapter A1 story — engineer awake at 2am, 35 minutes in, still hunting Confluence, ends up reading a six-month-old Jira ticket — that's a runbook failing in the most ordinary way imaginable.

Now the honest part. The thing agents are bad at.

I'd be a poor architect if I only sold you the upside.

An agent can be confidently wrong.

A script that fails, fails loudly. Stack trace. Non-zero exit. You know instantly.

An agent that's wrong produces a fluent, well-structured, professional-sounding answer that happens to be incorrect — and it has no idea. It cannot tell the difference between knowing and guessing.

Everything in Volume 3 exists because of that one sentence.

And it's why your product does three specific things:

Confidence score on every analysis
LOW / MEDIUM / HIGH / CRITICAL risk label on every command
Human approval required before anything dangerous runs

Those aren't features somebody added for fun. They are the direct, necessary consequence of this limitation. When a customer asks "how do I know it's right?" — that's the honest answer: you don't, so we built controls around it.

That answer earns more trust than any claim of accuracy ever will.

Your turn — six questions

Answer in the chat. Don't scroll back up. Rough, in your own words is exactly what I want — I need to hear how you're thinking, not have my words repeated back.

In your own words — what's the difference between a script and an agent?

What is the only thing an LLM can actually do?

If an LLM can't execute anything, how does your agent SSH into a server?

Name the three ingredients of an AI agent.

A customer asks: "Why not just use ChatGPT? Why pay you?" — What do you say?

What's the single biggest weakness of an AI agent — and how does your product handle it?

Answer those and I'll tell you where you're sharp and where you're fuzzy. Then Lesson 2, where we take one real alert and walk it through your product — 02:00:00 to 02:01:35, every single step.
