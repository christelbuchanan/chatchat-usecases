# Launch a ChatChat agent: the actionable connector playbook

This is the deployment manual for going from **one useful agent today** to a connected,
proactive personal or team operating system. It is deliberately ordered by maturity:

1. create one agent with one job;
2. connect one source and one destination;
3. prove the workflow manually;
4. turn on event-driven notices or a scheduled brief;
5. add shell computation and a second destination;
6. graduate only trusted, reversible steps toward automation.

The examples are based on capabilities visible in this backend. Provider availability,
permissions, write scopes, plan access, and regional support can vary. Preview the
connector in the product and test with non-sensitive data before relying on a workflow.

## The 15-minute first deployment

### 1. Choose an outcome, not a personality

Start with one sentence:

> “This agent helps **[person/group]** notice **[signal]**, turn it into **[artifact or
> decision]**, and deliver it to **[place]**, while never **[boundary]**.”

Examples:

- “My Briefing Agent notices commitments in meeting notes, updates a Google Doc, and gives
  me a weekday briefing, while never sending messages without approval.”
- “Our Run Club Agent combines shared plans and approved fitness summaries, posts a Sunday
  Telegram recap, and never reveals anyone's private health brain.”
- “My Research Agent turns saved notes and CSVs into a sourced report and slide deck, while
  never presenting an inference as a sourced fact.”

### 2. Create the agent

In the ChatChat product, create a new agent and configure:

1. **Name:** role-based and legible, such as “Weekly Architect” or “Impact Curator.”
2. **Instructions:** mission, audience, inputs, outputs, tone, approval gates, forbidden
   actions, and what should be remembered.
3. **Brain:** add only the documents and memories needed for this job.
4. **Connectors:** begin with one input and one output from the combinations below.
5. **Proactivity:** leave broad notifications off until the manual workflow is useful.
6. **Test:** ask the agent to show its sources and draft the output without taking action.

The backend creation contract is implemented by
[`POST /agents/save`](../src/web/features/agent/agent.controller.ts), with fields defined by
[`CreateAgentDto`](../src/shared/dtos/agent.dto.ts) and the
[`Agent` schema](../src/shared/schemas/agent.schema.ts). An authenticated API client can use
the same contract:

```http
POST /agents/save
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Weekly Architect",
  "systemPrompt": "Build a calm weekly plan. Cite source events, expose conflicts, preserve margin, and ask before changing calendars or sending messages.",
  "avatar": "https://example.com/weekly-architect.png",
  "proactiveSettings": {
    "proactiveEnabled": true,
    "quietHours": {
      "start": "21:00",
      "end": "07:00",
      "timezone": "Europe/London"
    },
    "notificationPreferences": {
      "urgentOnly": false,
      "dailyDigest": true,
      "dailyDigestTime": "07:30"
    }
  }
}
```

Do not paste production tokens into a document or ordinary chat. The HTTP example is for
developers using an authenticated client; most people should use the product interface.

### 3. Give the agent an operating contract

Copy and adapt this:

```text
ROLE
You are my Weekly Architect.

OUTCOME
Help me enter each week with clear priorities, realistic travel time, and two blocks of margin.

SOURCES
Use my Compass brain, Google Calendar, Gmail commitments, and Google Tasks. Cite the source
for a deadline or promise. Do not use private health or relationship memories.

OUTPUT
Produce one page: fixed commitments, conflicts, proposed focus blocks, unowned work, and two
things to defer. Label facts, calculations, and recommendations separately.

PROACTIVITY
Send the normal brief Sunday at 18:00 Europe/London. Interrupt outside that brief only for a
deadline within 24 hours, an impossible travel transition, or a blocked person.

APPROVAL
Draft freely. Ask before sending, posting, booking, paying, deleting, publishing, or changing
an event or task. Show destination and audience at approval time.

MEMORY
Remember stable preferences and approved final decisions. Do not retain incidental private
details, credentials, or raw data after the artifact is complete.
```

## What an agent can do today

### Converse and remember

An agent follows its system prompt and can use scoped brain context across conversations.
Design separate brains for separate trust boundaries rather than giving one agent universal
access. Brain capture and bindings are exposed under the agent-scoped brain controllers in
[`src/web/features/brain`](../src/web/features/brain).

### Read and act through connectors

Connected functions are registered centrally in the
[`IntegrationRegistry`](../src/web/features/integrations/integration.registry.ts). Depending
on connector scopes, an agent can search and read records, create or update artifacts, and
prepare or perform write actions. The integration API exposes available connections and
agent functions through the
[`IntegrationController`](../src/web/features/integrations/integration.controller.ts).

Treat “connected” and “proactively streaming events” as different states. A connector can
be callable in chat without supporting background sync, and sync-capable connectors still
need syncing enabled for the relevant agent connection.

### Notice events proactively

Sync-capable connectors produce normalized integration events. The analysis worker compares
events across sources and deliberately looks for noteworthy, urgent, or actionable patterns,
including cross-connector patterns. It creates bell notifications and injects an agent turn
into chat with a proposed next step. See the event-analysis rules in
[`analysis.processor.ts`](../src/worker/analysis/processors/analysis.processor.ts).

The current repo marks these connectors as sync-capable: **Gmail, Google Calendar, Google
Docs, Google Forms, GitHub, Jira, Microsoft 365, Notion, Oura, Plaid, Slack, Strava, Terra,
Wahoo, Whoop, Zoom**, plus health sources implemented on Terra-derived or event-producing
paths. Confirm the live connector's Sync control in the product; this list will evolve.

**Event-streaming recipe**

1. Connect the service to the specific agent.
2. Enable Sync for that connection if the connector supports it.
3. Enable proactive notifications on the agent.
4. Set quiet hours and urgency preferences.
5. Generate a harmless provider event.
6. Confirm the notification cites the correct connector and proposes rather than silently
   performing a consequential action.
7. Add a second source only after the first source has an acceptable signal-to-noise ratio.

The heartbeat enqueues due connector syncs and scheduled tasks; production uses a Kubernetes
CronJob and development has a one-minute internal heartbeat. That behavior is documented in
[`HeartbeatService`](../src/web/features/proactive/heartbeat.service.ts).

### Run scheduled tasks without writing code

Users can describe a custom recurring instruction, timezone, and daily, weekly, or monthly
schedule. They do **not** need to write a cron expression or program. The API can also
enhance a rough instruction, preview the result without delivery, trigger a test run, and
enable, disable, update, or delete the task. See
[`ScheduledProactiveController`](../src/web/features/scheduled-proactive/scheduled-proactive.controller.ts)
and the validated schedule shape in
[`scheduled-proactive-task.dto.ts`](../src/shared/dtos/scheduled-proactive-task.dto.ts).

Example custom task:

```http
POST /agents/<agentId>/scheduled-tasks
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Sunday family load balancer",
  "taskType": "custom",
  "prompt": "Using the last 7 days of approved calendar and task context, prepare next week's plan by person and day. Flag collisions and unowned work. Suggest a fair redistribution. Do not change calendars.",
  "schedule": {
    "frequency": "weekly",
    "interval": 1,
    "daysOfWeek": [0],
    "timeOfDay": "18:00"
  },
  "timezone": "Europe/London",
  "options": {
    "lookbackHours": 168,
    "integrationIds": ["google-calendar"]
  }
}
```

Then call `POST /agents/<agentId>/scheduled-tasks/<taskId>/trigger` to test it now.

> **Important verified boundary:** the current scheduled-proactive worker gathers recent
> events and conversation context, asks the model to compose one proactive message, and
> delivers that message into chat. It does **not** enter the normal tool loop and therefore
> does not directly execute connector functions or shell commands. This is visible in
> [`scheduled-proactive.processor.ts`](../src/worker/scheduled-proactive/processors/scheduled-proactive.processor.ts).
> A scheduled prompt can tell the user that a report is due and invite them to run it, but
> documentation should not claim that the backend scheduler itself created a spreadsheet,
> ran Python, or posted to Telegram.

### Use the terminal and shell as a workbench

During an interactive agent run, `sandbox_execute` can execute Python or Bash for data
analysis, calculations, charts, scripts, scraping, and document or file conversion. It can
optionally install Python packages and has access to uploaded files. The authoritative tool
description is in
[`chat-tool-catalog.service.ts`](../src/web/features/chat/chat-tool-catalog.service.ts).

The shell makes the practical artifact surface extremely broad, subject to installed tools,
package availability, execution timeouts, and the user's files:

| Ask for | Typical shell approach | Deliverable |
| --- | --- | --- |
| Excel workbook | Python with `openpyxl`, `xlsxwriter`, or `pandas` | `.xlsx` with formulas, tables, and charts |
| PowerPoint deck | Python with `python-pptx` | `.pptx` briefing or presentation |
| PDF compression | `ghostscript`, `qpdf`, or another available PDF tool | Smaller `.pdf`, verified for readability |
| PDF extraction/merge | Python PDF libraries or installed CLI tools | Text, tables, or combined PDF |
| Charts and dashboards | `pandas`, `matplotlib`, Plotly, or HTML | PNG, SVG, HTML, or embedded workbook chart |
| Data cleanup | Python, `jq`, `awk`, or CSV tools | Clean CSV/JSON/XLSX plus reconciliation report |
| Audio/video conversion | `ffmpeg` when available | Transcoded or compressed media |
| Repository analysis | `git`, `rg`, test and lint commands | Evidence-backed technical brief |
| Repeatable automation | Versioned Python or shell script | Re-runnable workflow in the workspace |

ChatChat also exposes dedicated `image_generate` and `image_edit` tools for raster visuals;
charts remain a shell/data-analysis job. Google Slides and Canva can be used as connected
destinations, while a shell-generated PowerPoint or Excel file is an artifact rather than a
live provider document.

**Artifact prompt**

```text
Use the attached CSV to create an executive workbook and a 7-slide PowerPoint.
1. Validate types, missing values, duplicates, totals, and date coverage.
2. Save the cleaning script and assumptions beside the outputs.
3. Put detailed reconciliations in Excel and only decision-relevant charts in the deck.
4. Cite the source filename and analysis timestamp.
5. Verify that both generated files reopen and contain the expected sheets/slides.
```

Persistent files under the managed runtime's home can survive later runs, but ordinary shell
process state does not. Use the persistent runtime only for workflows that genuinely require
saved CLI authentication, browser state, long-running processes, or continuity.

### Generate and edit images

The agent has dedicated image-generation and image-editing tools for posters, concepts,
mockups, banners, diagrams, and transformed uploads. Canva, Figma, Higgsfield, Instagram,
TikTok, YouTube, LinkedIn, and X can extend the workflow into design, video, review, or
distribution where the account has the required permissions. Generation does not imply
publication: review visual claims, likenesses, brand rules, destination, and audience first.

### Collaborate in Telegram

Connect **Telegram (Bot)** to an agent, create or select a bot through Telegram's BotFather,
and add that bot to a group. The worker supports real agent chat turns and attachment
preparation, while delivery is handled through a dedicated reliability path. The relevant
implementation lives in
[`src/worker/telegram`](../src/worker/telegram) and
[`telegram-bot`](../src/web/features/integrations/telegram-bot).

**Safe group setup**

1. Give the bot only the Telegram permissions it needs.
2. Add it to a test group before a real community.
3. Define when it speaks: on mention, on an explicit command, or at an agreed ritual.
4. Bind it to a shared brain; do not let group answers reach into private personal brains.
5. Test text and supported attachments.
6. Tell the group what the agent remembers and how to correct or remove information.
7. Use it for decisions, summaries, FAQs, logistics, and follow-ups—not covert monitoring.

Example group instruction:

```text
You are a participant in the Atlas trip group. Reply when mentioned. Use only the Atlas
shared brain and this group's messages. Summarize decisions with owner and date. Never reveal
private chats or personal-brain memories. Ask before posting a long recap or sharing a file.
```

## Connector catalog: what is available

The repository currently contains integrations across the following groups. Exact functions
and authorization requirements are visible through connector Preview in the product.

- **Communication:** Gmail, Microsoft 365, Slack, Telegram, Telegram Bot, Instagram,
  LinkedIn, TikTok, X, YouTube, Zoom.
- **Documents and productivity:** Google Calendar, Docs, Forms, Sheets, Slides, Tasks,
  Notion, GitHub, Jira, Figma, Canva, HubSpot, Granola.
- **Health and fitness:** Clue, Garmin, InBody, MyFitnessPal, Oura, Rouvy, Strava, Terra,
  Ultrahuman, Wahoo, Whoop.
- **Finance and commerce:** Airwallex, Binance, Plaid, Stripe, TikTok Business.
- **Events, location, and mobility:** Luma, Google Maps, Places, geocoding, Uber, Grab.
- **Creative media:** Canva, Figma, Higgsfield, Instagram, TikTok, YouTube.

## Starting combinations: deploy these first

Starting combinations use one source, one destination, and usually one approval point.

### 1. Gmail + Google Tasks: the promise catcher

**Deploy:** Connect both; manually ask the agent to find explicit commitments in a selected
thread and propose tasks with source, owner, and date. Approve the tasks individually.

**Then schedule:** A weekday reminder can summarize recent Gmail events and ask which
candidate commitments should become tasks. It will not create them until an interactive run
uses the Tasks connector.

**Prompt:** “Find promises I made in these emails. Do not interpret politeness as a promise.
Draft tasks with a source link and ask before creating them.”

### 2. Google Calendar + Google Docs: tomorrow's briefing

**Deploy:** Ask for tomorrow's meetings, retrieve approved brain context, and draft a single
Google Doc containing purpose, people, open decisions, and preparation questions.

**Proactive option:** Enable Calendar sync for time-sensitive meeting notices; add a daily
scheduled message that tells you when the briefing ritual is due.

### 3. Granola or Zoom + Notion: meeting memory

**Deploy:** Pull one transcript or note, extract decisions and unresolved questions, and
append the reviewed result to a Notion project database.

**Boundary:** Keep the transcript as the source, distinguish a decision from a suggestion,
and confirm database and page before writing.

### 4. Oura or Whoop + Telegram Bot: the humane morning check-in

**Deploy:** Read one recovery source, combine it with a subjective Telegram check-in, and
return a short reflection. Never diagnose or let a device score dictate the day.

### 5. CSV upload + shell + Google Sheets: the analyst in chat

**Deploy:** Upload a CSV, ask the agent to validate and clean it with Python, save the script,
then write or upload the reviewed summary to Sheets. This proves computation before adding
more live data sources.

### 6. Telegram Bot + shared brain: the group memory keeper

**Deploy:** Add the agent to a test group and answer only on mention. Ask it to summarize one
thread into decisions, owners, and dates, then correct the result in the group.

## Intermediate combinations: close a real loop

Intermediate combinations add a second signal, a proactive trigger, or a multi-artifact
handoff.

### 1. Granola/Zoom + Jira + Google Docs + Slack: meeting to execution

1. Retrieve meeting notes.
2. Draft the decision record in Docs.
3. Propose Jira issues with owner and acceptance criteria.
4. Prepare a concise Slack recap linking both artifacts.
5. Ask for one approval bundle before writes.
6. Later, let Jira and Slack sync surface deadline or ownership drift.

### 2. Gmail + Calendar + Maps + Uber/Grab: realistic travel guardian

The agent detects a location change, checks the calendar transition, calculates route and
buffer, and proposes leave-time or rescheduling options. The user approves any calendar edit,
ride booking, or spend.

### 3. GitHub + Jira + shell + Google Slides: release commander

Retrieve issues and pull requests, use the shell to inspect diffs or run targeted tests,
reconcile readiness, and create a short launch deck. Keep deployment and rollback actions
human-approved.

### 4. HubSpot + Gmail + Stripe: customer-risk radar

Combine CRM stage, communication change, and commercial context. Stream only material events;
prepare an evidence-backed account brief and a draft recovery message. A human reviews the
interpretation and outreach.

### 5. Oura/Whoop + Strava + Calendar + shell: interpreted-self lab

Align recovery, training, travel, and schedule data in Python; mark missing data and
confounders; generate a weekly chart and three bounded hypotheses. Store the reviewed result
in Notion or Docs, not every raw datapoint in a universal brain.

### 6. Notion + fitness connectors + shell: a personal fitness database

This mirrors a compelling real-world pattern: use **Notion as the human-readable database**,
connect sources such as Oura, Whoop, Strava, Garmin, MyFitnessPal, or InBody, and let the
shell normalize their different time scales before updating a dashboard.

**Suggested Notion schema**

| Database | Key fields |
| --- | --- |
| Daily state | Date, sleep window, recovery/readiness, subjective energy, note, source links |
| Training | Start time, activity, duration, load, intensity, source, linked daily state |
| Experiments | Hypothesis, protocol, start/end, confounders, outcome, confidence |
| Weekly review | What changed, chart links, interpretation, next question |

**Deploy safely:** begin with a manual weekly import; use the shell to create a normalized
table and exception report; ask before updating Notion; avoid diagnosis and composite “life
scores.” Once trusted, sync-capable sources can feed proactive pattern notices.

### 7. Forms + Sheets + shell + Slides: research synthesis engine

Collect structured responses in Forms, store them in Sheets, analyze themes and segments in
Python, and build a reviewed Slides narrative. Preserve response privacy, small-cohort
anonymity, and links from every claim back to evidence.

## Advanced and unexpected combinations

Advanced combinations coordinate domains. They require explicit brain boundaries,
audience-aware outputs, failure behavior, and an audit trail.

### 1. Calendar + recovery + Maps + weather-aware planning: the adaptive day

Combine a coarse recovery trend, meeting density, travel transitions, and location context.
Propose—not impose—a different workout, travel buffer, or focus block. This is valuable
because each connector changes the meaning of the others.

### 2. Plaid + Calendar + Compass brain + Sheets: the sabbatical simulator

Use read-only financial context and stated definitions of “enough,” then use the shell to
model ranges for dates, spending, and income assumptions. Place formulas in a workbook and
produce a decision memo. Do not recommend products, trade, transfer, or manufacture certainty.

### 3. GitHub + professional impact brain + LinkedIn + image generation

Detect an approved shipped outcome, connect it to project context and collaborators, draft a
case study, generate a diagram or visual, and prepare a LinkedIn post. Review confidentiality,
credit, claims, image rights, and audience before publication.

### 4. Luma + Forms + Maps + Telegram + Canva: the event operating system

Turn registrations into a consent-aware guest brief, flag access or dietary needs, plan
routes, create reviewed event assets, and use a Telegram group for live changes. Delete
temporary sensitive logistics after the event.

### 5. Family Telegram + Calendar + school PDFs + shell + Tasks

Group messages and school emails supply the inputs; the shell extracts dates and tables from
awkward PDFs; the agent proposes Calendar events and Tasks; the family group negotiates
ownership. The crucial output is not a longer list—it is visibly redistributed work.

### 6. Clue + Oura/Ultrahuman + MyFitnessPal + private journal

Explore whether cycle, sleep, temperature, nutrition, and subjective context move together.
Use a private brain, minimum retention, confidence labels, and bounded experiments. The
system supports questions for a qualified professional; it does not diagnose.

### 7. Stripe/Airwallex + HubSpot + Calendar + Telegram: founder control tower

Prepare a private morning exception brief covering cash movement, pipeline risk, customer
commitments, and today's decision windows. Use Telegram for the private output, not a broad
team group; financial action and customer outreach remain approved.

### 8. Creative fragments + Figma/Canva + Higgsfield + social connectors

Capture ideas through a private Telegram bot, synthesize a creative brief in Docs, develop
visual directions, create media, and prepare variants for Instagram, TikTok, YouTube, X, or
LinkedIn. Separate the private studio from the distribution agent so an experiment cannot
accidentally publish itself.

### 9. Relationship promises + travel logistics + shared group memory

The agent notices your explicit promise to organize a reunion, checks attendee-approved
availability, uses Maps and Places for realistic options, creates a Luma event, and brings a
Telegram agent into the group. Private relationship context never enters the shared brain.

### 10. Quarterly Life OS audit

Receive compact summaries from separate Work, Health, Money, People, and Play brains. Use
the shell to regenerate transparent charts, build a private retrospective in Docs, and audit
every proactive workflow for value, false positives, permissions, and deletion. Refuse a
single life score.

## Scheduled-task combinations that are deployable now

These schedules produce proactive chat messages grounded in recent selected connector events.
They are actionable today without programming:

| Schedule | Event context | Message to produce |
| --- | --- | --- |
| Weekdays 07:30 | Gmail + Calendar | Important asks, meetings, conflicts, and one decision needed |
| Sunday 18:00 | Calendar + Tasks | Weekly load, collisions, unowned work, and proposed trade-offs |
| Friday 16:30 | GitHub + Jira + Slack | Shipped outcomes, blockers, evidence to preserve, and open promises |
| Daily 17:00 | Gmail + Slack | Commitments made today that need confirmation or a task |
| Monday 08:00 | Oura/Whoop + Strava | Weekly pattern summary with uncertainty and one safe question |
| Month-end | Plaid | Material changes and reconciliation questions for a human review |
| Before community ritual | Telegram/Slack | Shared decisions, unresolved questions, and named owners |

### How to link a schedule to shell work honestly

Today, use a **two-step approval pattern**:

1. Scheduled task posts: “The monthly close is due. I found 3 new Plaid event groups. Want
   me to reconcile them and generate the workbook now?”
2. User replies: “Yes, run it.”
3. The interactive agent run retrieves permitted data, calls `sandbox_execute`, validates the
   workbook, and returns the artifact.

This keeps a human in the loop and matches the current implementation. Do not describe this
as unattended cron-to-shell execution.

If a user explicitly asks the interactive agent to configure an **OS-level cron job** inside
a persistent managed runtime, the shell may be able to do so when the runtime, operating
system, installed scheduler, and persistence policy support it. That is machine-level
automation—not the ChatChat Scheduled Tasks feature—and it must be tested for survival across
runtime restarts. Never use it to bypass connector approvals or store secrets in crontabs.

**Future product opportunity:** route a scheduled task into a full chat/tool run with a
declared tool allowlist, artifact destination, approval policy, runtime budget, idempotency
key, and audit record. Until that exists in the worker path, scheduled terminal execution
should be labeled a roadmap pattern rather than a current capability.

## Custom MCPs and the expanding connector frontier

ChatChat already has an MCP client foundation and productized MCP-backed integrations such
as Airwallex, Granola, Higgsfield, Stripe, and TikTok Business. The base client and OAuth
support live under [`src/web/features/integrations/mcp`](../src/web/features/integrations/mcp).
General bring-your-own MCP installation should be treated as forthcoming unless it is
exposed and enabled in the live product.

When custom MCP support is available, useful external projects to evaluate include:

- [GitHub's official GitHub MCP Server](https://github.com/github/github-mcp-server) for
  repositories, issues, pull requests, and code workflows.
- [Microsoft Playwright MCP](https://github.com/microsoft/playwright-mcp) for browser
  automation where API connectors are unavailable.
- [Stripe Agent Toolkit](https://github.com/stripe/agent-toolkit) for Stripe agent tools and
  MCP patterns.
- [AWS Labs MCP Servers](https://github.com/awslabs/mcp) for AWS-oriented developer and
  operational workflows.
- [Model Context Protocol reference servers](https://github.com/modelcontextprotocol/servers)
  for examples and historical reference implementations.

These are examples to investigate, not a compatibility or security endorsement. Before
installing any MCP server, verify its current maintainer, license, transport, authentication,
tool descriptions, write actions, data retention, network reach, release history, and prompt-
injection defenses. Pin versions and begin read-only in a disposable environment.

**Unexpected MCP directions**

- a home-energy MCP + Calendar + weather data for load-shifting recommendations;
- a library or reading MCP + Kindle/exported notes + Telegram book club;
- an inventory MCP + purchase email + warranty brain for household asset management;
- a local-government MCP + Maps + Calendar for personalized civic deadlines;
- an observability MCP + GitHub + Jira for an incident historian and learning review;
- a research-database MCP + Docs + shell for a citation-preserving evidence studio.

## Production checklist

### Outcome and scope

- [ ] The agent has one named user or group and one measurable outcome.
- [ ] The system prompt states sources, outputs, tone, memory, and forbidden actions.
- [ ] Each connector has a reason to exist in the workflow.

### Trust and privacy

- [ ] Brains are separated by audience and sensitivity.
- [ ] The agent has the least connector permissions required.
- [ ] Shared chat responses cannot reach private memories.
- [ ] Health, finance, children, and relationship data have explicit retention rules.

### Proactivity

- [ ] Sync is enabled only on connectors intended to stream events.
- [ ] Quiet hours, materiality, and urgency are defined.
- [ ] The workflow has mute, correction, and deletion paths.
- [ ] Scheduled-task output is not misrepresented as unattended tool execution.

### Actions and artifacts

- [ ] Sends, posts, bookings, payments, deletes, deployments, and sensitive shares require
  approval.
- [ ] Shell scripts preserve assumptions and can be rerun.
- [ ] Generated PPTX, XLSX, PDF, images, and media are reopened or otherwise validated.
- [ ] A canonical source wins when two connected systems can edit the same truth.

### Graduation

- [ ] The manual version worked before proactivity was enabled.
- [ ] False positives and maintenance time are reviewed after two to four weeks.
- [ ] The workflow recovers attention or improves a decision more than it creates admin.
- [ ] There is an automatic or human stop condition when the workflow is no longer useful.

## Copy-ready deployment canvas

```text
AGENT NAME:
AUDIENCE:
ONE OUTCOME:

SOURCE BRAIN(S):
SOURCE CONNECTOR(S):
DESTINATION CONNECTOR(S):
SHELL/ARTIFACT WORK:

EVENT TRIGGER:
SCHEDULE + TIMEZONE:
QUIET HOURS:
MATERIALITY THRESHOLD:

MAY DO ALONE:
MUST ASK BEFORE:
MUST NEVER DO:

SHARED-CHAT VISIBILITY:
MEMORY TO KEEP:
DATA TO EXPIRE:

MANUAL TEST:
EXPECTED OUTPUT:
SUCCESS MEASURE:
REVIEW/DELETE DATE:
```

The best first deployment is not the most connected one. It is the smallest loop that
reliably notices something useful, makes the reasoning inspectable, proposes a proportionate
next step, and leaves the person more capable than before.
docs/chatchat-lifemaxxing-plays.md
docs/chatchat-lifemaxxing-plays.md
New
+829
-0

# 10 plays for life-maxxing with ChatChat

Life-maxxing should not mean squeezing productivity from every minute. Here it means
building a life with more energy, agency, connection, security, growth, and room to be
human. ChatChat can reduce coordination work, notice useful patterns, and help intentions
survive a busy week—but it should not decide what a good life means for you.

Each play combines five layers:

- a **Living Brain** that remembers the right context;
- an **agent** with one clear responsibility;
- **integrations** that connect real signals and destinations;
- a **proactive workflow** that notices only meaningful changes;
- the **shell** for transparent calculation, file processing, and repeatable analysis.

Start with one play. Run it for four weeks, review what helped, and delete what merely
created more admin.

## Before you begin: build a life portfolio, not one giant brain

Use separate brains for separate purposes and trust boundaries:

| Brain | Remember | Avoid |
| --- | --- | --- |
| **Compass** | Values, current season, priorities, constraints, definitions of “enough” | Other people's expectations presented as your goals |
| **Health** | Habits, experiments, subjective notes, relevant metrics | Diagnoses, unsupported conclusions, unnecessary medical detail |
| **Work and craft** | Projects, decisions, evidence of impact, skills, feedback | Confidential information you are not permitted to store |
| **Money** | Goals, budgets, assumptions, review decisions | Passwords, recovery codes, or unrestricted account credentials |
| **People** | Commitments, shared memories, ways to show care | Gossip, covert scoring, or inferred sensitive traits |
| **Play and possibility** | Curiosities, adventures, creative sparks, “someday” ideas | Turning every hobby into an outcome metric |

Let a private **Life OS agent** read short summaries from these brains. Do not give every
specialist agent access to every brain. Share the minimum context needed for the immediate
job and expire temporary handoffs.

## Play 1: write a one-page life compass

Most systems optimize whatever is easiest to count. Start by telling the system what
actually matters in this season of life.

**Set it up**

- Create a Compass brain with five headings: what matters, what “enough” looks like,
  non-negotiables, current constraints, and what you are intentionally not pursuing.
- Give a **Compass agent** permission to challenge calendar or task patterns, but not to
  change them automatically.
- Connect Calendar and Tasks. Once a month, use the shell to categorize a calendar export
  and compare actual time with stated priorities.

**Proactive move:** Send a monthly alignment note only when a priority is repeatedly
crowded out or a “not now” commitment is quietly consuming time.

**Try:** “Help me define a good next 90 days across energy, people, work, money, and play.
Ask one question at a time. Turn my answers into principles, not an impossible task list.”

**Measure:** Fewer accidental commitments and a clearer “no,” not a perfect alignment
score.

## Play 2: reclaim the week before it claims you

Create a weekly control tower that resolves collisions before Monday and preserves margin
for surprises.

**Set it up**

- Give a **Weekly architect agent** access to the Compass summary, Calendar, Gmail-derived
  commitments, and Google Tasks.
- Have it distinguish fixed events, movable work, care responsibilities, recovery, and
  unallocated margin.
- Use a shared Telegram chat when a household or team needs to negotiate the plan together.

**Proactive move:** On Sunday, propose a one-page week with conflicts, overloaded days,
unowned commitments, and two things to drop. Alert midweek only when reality materially
diverges from the plan.

**Try:** “Design next week around my three priorities and actual energy. Preserve two
blocks of margin, show every collision, and ask before moving or declining anything.”

**Measure:** Number of preventable collisions and last-minute scrambles, not calendar
utilization.

## Play 3: build an energy manual for yourself

Replace generic wellness advice with careful observation of your own patterns. This is a
reflection tool, not medical diagnosis or treatment.

**Set it up**

- Keep sleep, training, nutrition, cycle, travel, and subjective check-ins in a scoped
  Health brain.
- Connect relevant sources such as Oura, Whoop, Garmin, Strava, Ultrahuman, MyFitnessPal,
  InBody, Clue, or Terra.
- Let a **Health experiment agent** use Python in the shell to align time series, chart
  trends, and record assumptions and confounders.

**Proactive move:** Suggest a lighter day when several agreed signals align, or surface a
pattern for review after it repeats—never issue an alarm from a single noisy datapoint.

**Try:** “Compare sleep regularity, training load, travel, and my 1–5 energy notes for the
last eight weeks. Describe correlations, uncertainty, and three hypotheses I could safely
test; do not diagnose.”

**Measure:** Better questions, more consistent energy, and experiments completed—not
chasing a device score.

## Play 4: turn promises into trust

Use an agent to remember commitments so attention can return to the person in front of
you.

**Set it up**

- Create a scoped People brain for appropriate relationship context and explicit promises.
- Give a **Promise keeper agent** access to relevant Gmail, Calendar, Slack, Telegram, and
  meeting notes.
- Require a source link for every captured commitment and keep private and group memories
  separate.

**Proactive move:** Nudge before a promise becomes late, when the recipient still has time
to benefit. Offer “do, delegate, renegotiate, or drop” rather than silently creating tasks.

**Try:** “Find commitments I made this week. Quote the source briefly, identify the person
and date, and ask me which ones should become tasks. Do not message anyone.”

**Measure:** Promises closed or thoughtfully renegotiated, not follow-up volume.

## Play 5: create a personal board of advisors

Better decisions often require multiple lenses rather than one authoritative-sounding
answer.

**Set it up**

- Create three agents: a **pragmatist** for constraints and reversibility, a **possibility
  scout** for upside and experiments, and a **values guardian** grounded in the Compass.
- Give all three the same decision brief, evidence, deadline, and known unknowns.
- Ask a **Chair agent** to synthesize disagreements without forcing consensus.

**Proactive move:** When the decision date approaches, identify missing evidence and the
smallest reversible test. Do not repeatedly reopen a settled decision unless an assumption
changes.

**Try:** “Run this decision through my three advisors. Show where they agree, where they
disagree, which assumption drives each view, and the cheapest way to learn more.”

**Measure:** Decision speed, reversibility, and quality of post-decision learning—not the
number of options generated.

## Play 6: compound your professional evidence

Do not wait for a review, promotion, proposal, or job search to reconstruct a year of
impact from memory.

**Set it up**

- Keep an Impact brain with outcomes, metrics, attributed feedback, difficult problems,
  lessons, and source links.
- Let an **Impact curator agent** scan approved project updates, GitHub, Jira, Gmail, Slack,
  and meeting notes for candidate evidence.
- Use the shell to reconcile project or calendar exports and produce simple trend charts;
  let Google Docs or Slides hold the human-readable portfolio.

**Proactive move:** Every Friday, propose three evidence entries and ask what is missing.
Before a review, produce a role-relevant narrative that separates outcomes from activity
and includes often-invisible work such as mentoring or incident support.

**Try:** “Draft this month’s impact record with evidence links. State the problem, my
specific contribution, result, and lesson. Mark uncertain attribution instead of guessing.”

**Measure:** Evidence captured close to the event and fewer forgotten contributions—not
self-promotion frequency.

## Play 7: automate financial calm, not financial obsession

Create a predictable review rhythm that surfaces genuine exceptions while keeping money
decisions human. This workflow supports organization and scenario planning, not financial
advice.

**Set it up**

- Record goals, definitions, and scenario assumptions in a Money brain; never store
  passwords, MFA codes, or recovery secrets.
- Give a **Money review agent** read-only access to appropriate Plaid, Stripe, Airwallex, or
  Google Sheets data.
- Use the shell to normalize exports, check duplicates, calculate ranges, and retain the
  formulas used for next month.

**Proactive move:** Alert for an agreed material variance, unusual recurring charge, or
upcoming cash constraint—not every transaction. Require approval for transfers, purchases,
or changes to financial systems.

**Try:** “Reconcile these monthly exports, explain every assumption, show changes above my
materiality threshold, and model three scenarios. Do not move money or recommend products.”

**Measure:** Time spent worrying or reconciling, surprises caught, and decisions made with
clear assumptions—not net-worth refreshes.

## Play 8: build a curiosity-to-craft engine

Turn scattered interests into small finished artifacts without making joy another job.

**Set it up**

- Send articles, voice notes, screenshots, and questions to a Play and possibility brain,
  optionally through a private Telegram bot.
- Give a **Creative gardener agent** the job of connecting ideas and proposing one tiny
  artifact: an essay, prototype, playlist, recipe, illustration, or experiment.
- Use the shell for research-file cleanup, lightweight prototypes, conversions, or data
  visualization; use Canva, Figma, or Higgsfield when a visual artifact fits.

**Proactive move:** Once a week, resurface one idea that still feels alive and offer a
30-minute next step. Archive ideas that repeatedly feel like obligations.

**Try:** “Connect these five notes into three possible creative experiments. Optimize for
delight and learning, keep each under 90 minutes, and do not suggest monetizing them.”

**Measure:** Curiosity sustained and artifacts enjoyed—not audience size or side-hustle
revenue.

## Play 9: design deeper connection rituals

Use memory to help you show up with care, never to simulate intimacy or covertly profile
people.

**Set it up**

- Store only appropriate shared context: important dates, promises, interests someone has
  openly shared, and ideas for time together.
- Give a **Connection agent** access to the relevant People-brain slice and Calendar.
- Use a family or friends Telegram group for consensual plans and shared recaps, while
  keeping private one-to-one context out of group answers.

**Proactive move:** Notice when an important relationship has had no intentional time, a
friend’s meaningful date approaches, or a promised introduction remains open. Suggest a
genuine action; never auto-send a personal message.

**Try:** “Help me plan a thoughtful catch-up with Sam. Use only things Sam has shared with
me, remind me what I promised last time, and suggest three questions that invite—not
assume—what is happening in their life.”

**Measure:** Intentional time and promises honored—not contact cadence or a relationship
score.

## Play 10: run a quarterly life retrospective

Close the loop across all nine plays without collapsing life into a dashboard.

**Set it up**

- Let the private Life OS agent receive compact, permissioned summaries from each domain
  brain—not their raw archives.
- Use Calendar, Tasks, selected health trends, the impact portfolio, money scenarios, and
  personal notes as evidence.
- Let the shell generate transparent charts or comparisons, then let the agent build a
  reflective Google Doc with source links and unanswered questions.

**Proactive move:** At quarter end, prepare a quiet review with what gave energy, what
drained it, who mattered, what changed, what was learned, and which commitments should end.

**Try:** “Create my quarterly retrospective. Separate evidence from interpretation, name
the trade-offs I made, celebrate what cannot be graphed, and recommend one stop, one start,
and one continuation for the next season.”

**Measure:** Honest course corrections and commitments released—not an overall life score.

## The deeper thesis: life-maxxing is an orchestration problem

The first era of personal software asked people to become better database operators: fill
in the fields, maintain the lists, reconcile the dashboards, and remember which app held
which part of life. The first era of AI added a conversational layer, but often left the
same burden intact. You could ask a better question, yet you still had to assemble the
context, notice the moment, and carry the answer into the world.

Agentic life design points somewhere more interesting. The goal is not a single digital
oracle, and it is not maximum automation. It is a **small institution built around one
person or household**:

- brains provide institutional memory;
- connectors provide senses and hands;
- specialist agents provide accountable roles;
- proactive workflows provide timing;
- the shell provides a laboratory for reproducible reasoning;
- approval boundaries preserve agency.

This changes the unit of design. We stop asking, “What can the chatbot answer?” and start
asking, “What beneficial loop should continue even when I am busy?” A great loop notices
something worth noticing, assembles only the context it is allowed to use, proposes a
proportionate response, and remembers the outcome. It makes the person more present, not
more managed.

### The connector is not the strategy

Connecting everything is tempting and usually wrong. A connector is a capability, not a
reason to use it. Start with the life outcome, choose the smallest useful signal set, and
add a destination only after the insight consistently deserves action.

Think of the connector landscape in five layers:

| Layer | Examples available in ChatChat | Role in a play |
| --- | --- | --- |
| **Attention and commitments** | Gmail, Microsoft, Slack, Telegram, Telegram Bot, Zoom, Google Calendar, Google Tasks | Reveal asks, promises, meetings, and changes |
| **Knowledge and expression** | Google Docs, Sheets, Slides, Forms, Notion, Canva, Figma | Hold living artifacts and turn insight into something shareable |
| **Work and relationships** | GitHub, Jira, HubSpot, LinkedIn, Luma | Supply evidence about delivery, customers, professional ties, and events |
| **Body and movement** | Oura, Whoop, Garmin, Strava, Wahoo, Rouvy, Ultrahuman, MyFitnessPal, InBody, Clue, Terra | Support careful personal pattern-finding across recovery, training, nutrition, and cycles |
| **Money, place, and reach** | Plaid, Binance, Airwallex, Stripe, Google Maps, Places, geocoding, Uber, Grab, Instagram, TikTok, YouTube, X | Add financial context, physical-world logistics, distribution, and feedback |

Some integrations are mediated through MCP and connector capabilities can vary by account,
permission, region, and provider. Design graceful degradation: a play should still work
with a forwarded email, uploaded CSV, or manual check-in when a preferred connector is not
available.

### From dashboard to closed learning loop

A dashboard says what happened. A life-maxxing play should go further:

`signal → context → interpretation → proposed choice → approved action → observed outcome`

Every arrow deserves scrutiny. Which signal is reliable? Which brain may contextualize it?
What uncertainty should be visible? What action is reversible? What outcome would teach us
whether the play helped? Closing this loop is how a Living Brain becomes wiser instead of
merely larger.

## Expanded playbooks: from inspiring idea to operating system

The following field guides deepen each of the ten plays. The connector stacks are menus,
not installation checklists. Begin with the minimum viable version, earn trust, then add
reach.

### Play 1 expanded: the compass as a constitution

A life compass is not a vision board. It is a compact constitution for making trade-offs
when every request looks reasonable in isolation. It should describe the current season,
not legislate an identity forever.

**Connector stack**

- **Google Calendar** reveals where time actually went.
- **Google Tasks, Jira, or Notion** reveal the commitments competing for attention.
- **Gmail, Microsoft, and Slack** reveal incoming demand, but should initially contribute
  summaries rather than unrestricted message archives.
- **Google Docs** is a strong destination for the living compass; **Sheets** can hold a
  transparent monthly allocation table.

**Three-agent pattern**

1. A **Biographer** asks reflective questions and maintains the narrative of the current
   season.
2. An **Auditor** uses Calendar and task evidence to compare intention with behavior.
3. A **Constitutional guardian** reviews new opportunities against the compass and names
   the trade-off without making the decision.

**Shell laboratory:** Export calendar events to CSV, classify them with a reviewable rules
file, and produce a monthly chart. Keep the category mapping beside the script so changes
in the chart can be distinguished from changes in the definitions.

**Advanced proactive workflows**

- Detect a third consecutive week in which a declared priority receives no protected time.
- Before accepting a large new commitment, generate a “what leaves?” memo.
- At the end of a season, identify which constraints changed and propose a compass rewrite
  rather than optimizing against an obsolete document.

**Failure mode:** The compass becomes another instrument of guilt. Counter this by recording
constraints, care work, rest, and serendipity as legitimate realities—not deviations from
an idealized plan.

**Graduation test:** You can explain a difficult “not now” decision with greater clarity,
and the system helped you see the trade-off without pretending there was one correct answer.

### Play 2 expanded: the week as a negotiated portfolio

A calendar is often treated as a container. In practice, it is a negotiation between
ambition, obligation, energy, geography, and other people. Weekly architecture makes those
negotiations visible before they become emergencies.

**Connector stack**

- **Google Calendar or Microsoft** supplies fixed commitments and candidate focus blocks.
- **Gmail, Slack, and Telegram** surface explicit promises and late-breaking changes.
- **Google Tasks and Jira** supply work that has not yet claimed calendar space.
- **Maps, Places, geocoding, Uber, and Grab** make travel time and location transitions real.
- **Oura, Whoop, or Ultrahuman** can provide a coarse recovery signal when the user wants it;
  never let a wearable unilaterally cancel a meaningful plan.

**Household variation:** Put the proposed week in a family Telegram group. The agent posts
collisions and unowned work, people negotiate in public, and the final ownership returns to
the Family brain. This distributes coordination rather than making one person the human API
between everyone else's calendars.

**Shell laboratory:** Calculate travel buffers, meeting density, fragmented-time ratios,
and unallocated margin. Produce an inspectable exception list—not a mysterious optimization
score.

**Advanced proactive workflows**

- Warn when a location change makes the next commitment physically implausible.
- Detect when focus time has been fragmented into unusable pieces.
- Generate a caregiver or colleague handover containing only the information relevant to
  the handover window.
- Offer a recovery plan after disruption: preserve, move, delegate, renegotiate, or drop.

**Failure mode:** The “perfect week” collapses on Tuesday and creates shame. Design for
replanning: the system should recover gracefully, protect the most important constraint,
and abandon sunk plans without moral judgment.

**Graduation test:** Fewer preventable collisions, fairer ownership, and more usable margin.

### Play 3 expanded: the quantified self becomes the interpreted self

Health data has an authority aesthetic: precise numbers can look like truth even when the
measurement is noisy and the causal story is unknown. The better opportunity is not total
quantification; it is a patient dialogue between signals, felt experience, and context.

**Connector stack**

- **Oura, Whoop, Garmin, Ultrahuman, and Terra** can contribute sleep and recovery signals.
- **Strava, Wahoo, and Rouvy** add training context.
- **MyFitnessPal and InBody** can add nutrition and body-composition context.
- **Clue** can add cycle context when explicitly desired.
- **Calendar** adds travel, late meetings, and schedule disruption; a private Telegram bot
  can collect a ten-second subjective energy note.

**Evidence ladder**

1. raw observation: “sleep midpoint moved 90 minutes”;
2. repeated association: “lower energy followed on 6 of 8 comparable days”;
3. plausible hypothesis: “schedule variability may contribute”;
4. bounded experiment: “hold wake time steady for two weeks”;
5. human or professional interpretation where appropriate.

The agent must not jump from rung one to rung five.

**Shell laboratory:** Normalize time zones, align daily and workout data, mark missingness,
plot rolling distributions, and run simple sensitivity checks. Save a data dictionary and
analysis notebook so the result can be reproduced.

**Advanced proactive workflows**

- Surface a pattern only after a user-chosen recurrence threshold.
- Prepare a concise, source-linked question list for a qualified clinician or coach.
- Detect that an experiment has become confounded by travel or illness and recommend
  extending or abandoning it rather than declaring a result.

**Failure mode:** Metric anxiety. Support device-free weeks, hide routine scores, and let
subjective experience overrule an optimization loop.

**Graduation test:** The play produces calmer, better-informed experiments and better
questions—not more compulsive checking.

### Play 4 expanded: a commitment graph for human trust

Most task systems begin after someone has already translated a conversation into a task.
The trust opportunity begins earlier: detect the social commitment, preserve its source and
recipient, and keep the promise connected to the relationship that gives it meaning.

**Connector stack**

- **Gmail, Microsoft, Slack, Telegram, Zoom, and Granola via MCP** reveal commitments in
  communication and meetings.
- **Google Calendar** holds date-bound promises.
- **Google Tasks, Jira, HubSpot, or Notion** provide the appropriate execution destination.
- **Google Docs** can hold a shared decision or handover artifact.

**Commitment schema:** Store who promised what to whom, the source, due date confidence,
visibility, current state, and next review. Do not infer a promise from vague conversational
politeness without asking.

**Advanced proactive workflows**

- Find promises made by others that block your promise, and prepare a respectful unblock
  request.
- Notice that a commitment changed in one channel but remains stale elsewhere.
- Produce a Friday “trust close” with promises completed, renegotiated, waiting, or no
  longer valuable.
- In a Telegram group, report only group-visible commitments; never enrich them with private
  one-to-one context.

**Shell laboratory:** Reconcile exported task lists, identify duplicate or contradictory due
dates, and generate an aging report whose rules are plain text and reviewable.

**Failure mode:** Surveillance masquerading as reliability. Capture your own promises by
default; capture other people's only in an agreed shared workflow.

**Graduation test:** More honest renegotiation and fewer surprised people, not a larger task
inventory.

### Play 5 expanded: decision intelligence without synthetic certainty

A personal board of advisors works because disagreement is information. Multiple agents
should not create theatrical personas; each should represent a defined decision lens and
show its evidence.

**Connector stack**

- **Docs and Notion** hold the canonical decision brief and decision journal.
- **Sheets** supports scenario tables; the **shell** supports sensitivity analysis.
- **Gmail, Slack, HubSpot, LinkedIn, GitHub, and Jira** can provide stakeholder, customer,
  technical, and execution evidence when relevant and permitted.
- **Plaid, Stripe, Airwallex, or Binance** may contribute factual financial inputs, never
  autonomous investment or payment authority.

**Decision packet**

- the decision and decision owner;
- deadline and cost of delay;
- reversible versus irreversible components;
- options, including “do nothing”;
- evidence, assumptions, and missing information;
- stakeholders and consent boundaries;
- pre-mortem and smallest useful experiment.

**Advanced proactive workflows**

- Alert only when a named assumption changes before the decision date.
- Schedule a post-decision review at the time the outcome should become observable.
- Compare the forecast with the outcome and update calibration—not rewrite history.

**Shell laboratory:** Model scenario ranges rather than a single forecast, vary the decisive
assumptions, and produce a chart showing where the preferred option changes.

**Failure mode:** Agent consensus creates false confidence. Require each advisor to state
what would change its mind and let the Chair preserve unresolved disagreement.

**Graduation test:** Faster reversible decisions, slower irreversible decisions, and a
better record of why choices were made.

### Play 6 expanded: professional memory as career infrastructure

Organizations keep extensive memory about projects; individuals often keep almost none
about their own contribution. An impact brain redresses that asymmetry. It is not a vanity
archive—it is portable evidence for reflection, reviews, proposals, leadership growth, and
career transitions.

**Connector stack**

- **GitHub and Jira** contribute shipped work, reviews, incidents, and project movement.
- **Gmail, Microsoft, Slack, Zoom, and Granola** contribute decisions, feedback, and
  collaboration evidence.
- **HubSpot** can connect contribution to customer outcomes.
- **LinkedIn** can support a public professional narrative after review.
- **Docs, Slides, Notion, and Sheets** create portfolios, promotion cases, proposals, and
  quantified appendices.

**Evidence model:** Capture context, specific contribution, result, source, collaborators,
confidence, competency demonstrated, and lesson. Attribute shared wins generously. Keep
candidate entries private until reviewed.

**Advanced proactive workflows**

- Detect praise or outcome evidence and ask whether to preserve it.
- Connect invisible work—mentoring, onboarding, recruiting, incident support—to observable
  organizational outcomes.
- Compare Calendar allocation with role expectations and surface sustained mismatch.
- Before a review, generate multiple narratives for executive, craft, leadership, or
  customer-impact audiences without inventing metrics.

**Shell laboratory:** Analyze repository contribution carefully, avoiding crude line-count
rankings. Reconcile project exports, chart outcome trends, and retain source references.

**Failure mode:** Optimizing for legible work at the expense of valuable work. The compass
and role strategy should determine what matters; available connector data should not.

**Graduation test:** Reviews contain stronger evidence, invisible contributions become
visible, and career choices are grounded in a richer account of actual strengths.

### Play 7 expanded: a financial observatory, not an automated casino

Financial calm comes from clear definitions, predictable review, transparent assumptions,
and fewer surprises. More alerts and faster transactions often produce the opposite.

**Connector stack**

- **Plaid** can provide account-level context where supported.
- **Stripe and Airwallex via MCP** can provide business revenue and cash-flow context.
- **Binance** can contribute digital-asset data when relevant.
- **Gmail** can surface bills and receipts; **Sheets** can hold the reviewed model.
- **Docs or Slides** can produce a household, founder, or freelance monthly briefing.

**Three views**

1. **Stability:** upcoming obligations, cash buffer, and material anomalies.
2. **Direction:** progress against explicitly chosen goals.
3. **Possibility:** scenario ranges for a sabbatical, move, business investment, or other
   life choice—without pretending the future is forecastable.

**Advanced proactive workflows**

- Identify recurring costs that changed materially or outlived their stated purpose.
- Prepare a quarterly “money serves life” review connecting spending categories to the
  Compass without moralizing individual purchases.
- For founders or independents, model runway ranges when revenue or cost assumptions move.

**Shell laboratory:** Deduplicate transactions, normalize currencies and categories, version
formulas, and produce reconciliation exceptions. Never echo secrets or raw credentials.

**Failure mode:** Constant net-worth monitoring and false precision. Use materiality
thresholds, review windows, and ranges; keep transfers, trades, purchases, and account
changes behind explicit human action.

**Graduation test:** Shorter reconciliation, clearer choices, and fewer financial surprises.

### Play 8 expanded: a studio that protects aliveness

Generative tools can increase content volume while decreasing creative vitality. The more
interesting design is a studio that protects fragments, connects ideas across time, and
helps a person finish work that still feels like theirs.

**Connector stack**

- A private **Telegram Bot**, Gmail, or Forms captures fragments with low friction.
- **Docs and Notion** hold essays and research; **Sheets** can track experiments lightly.
- **Canva and Figma** support visual development; **Higgsfield via MCP** can support image
  generation where available.
- **Instagram, TikTok, YouTube, X, and LinkedIn** are optional destinations and feedback
  signals, not the creative brief.

**Studio agents**

- A **Gardener** connects fragments without demanding output.
- An **Editor** strengthens structure while preserving voice.
- A **Producer** turns an approved concept into channel-specific assets.
- An **Archivist** records what the creator learned, including work that was never published.

**Advanced proactive workflows**

- Resurface an old fragment when it connects meaningfully to a current question.
- Build a content atomization plan only after the core artifact is strong.
- Compare audience response qualitatively and quantitatively, while protecting the creator
  from making engagement the only definition of value.

**Shell laboratory:** Transcode files, clean transcripts, build contact sheets, analyze a
personal archive, and generate reproducible visualizations or prototypes.

**Failure mode:** The studio becomes a content treadmill. Set a creation-to-distribution
boundary, preserve private work, and include “made for no audience” as a valid outcome.

**Graduation test:** More finished artifacts that feel authentic, with less administrative
friction and no requirement to monetize joy.

### Play 9 expanded: technology that returns us to one another

The strongest relationship technology should disappear into better presence. It can carry
logistics and memory, but it cannot outsource curiosity, care, apology, or consent.

**Connector stack**

- **Calendar, Gmail, Microsoft, Slack, and Telegram** reveal shared plans and your explicit
  commitments.
- **Maps, Places, Uber, and Grab** reduce the logistical burden of getting together.
- **Luma and Forms** support gatherings and preferences with consent.
- **Docs** can preserve approved shared stories; a group Telegram agent can maintain group
  memory without accessing private brains.

**Relationship circles:** Define different memory and notification norms for household,
close friends, extended family, professional relationships, and community. Proximity does
not imply universal access.

**Advanced proactive workflows**

- Create a time-bounded trip or caregiver handover instead of exposing a whole brain.
- Before a gathering, surface access needs or preferences participants deliberately shared.
- After a community discussion, record decisions and ownership in the shared brain, then
  forget incidental personal details.
- Suggest a repair conversation when your own unfulfilled promise has aged; never generate
  emotional pressure on the other person.

**Shell laboratory:** Build fair rotations, compare availability, or deduplicate event lists.
Do not calculate relationship rankings or infer emotional states.

**Failure mode:** Simulated intimacy. Ban automated personal sending by default and make
source, visibility, and consent visible for remembered facts.

**Graduation test:** Less logistical friction and more genuine attention when people meet.

### Play 10 expanded: the retrospective as governance

A quarterly retrospective is where the Life OS becomes accountable. It asks whether the
other nine plays served the person, whether their permissions remain appropriate, and
whether some automations should end.

**Connector stack**

- Receive summaries—not raw universal access—from Compass, Calendar, Tasks, Health, Impact,
  Money, People, and Play brains.
- Use **Docs** for the reflective narrative, **Sheets** for inspectable supporting tables,
  and **Slides** only when sharing an approved version with a partner, coach, or team.
- Use the **shell** to regenerate charts from versioned scripts and snapshot the definitions
  used in the quarter.

**Retrospective agenda**

1. What felt alive, heavy, meaningful, or unexpectedly easy?
2. What does the evidence suggest, and what can it not tell us?
3. Which people and communities received real attention?
4. Which invisible constraints or forms of care shaped the quarter?
5. What did the agents get wrong, over-notify, or fail to notice?
6. Which memories should be corrected, archived, or deleted?
7. What should stop, start, continue, or remain deliberately unoptimized?

**Advanced proactive workflows**

- Produce an automation audit: value created, false-positive rate, permissions, sensitive
  memories, and last meaningful use.
- Compare prior forecasts and experiments with outcomes to improve calibration.
- Create a shareable summary containing only items explicitly approved for its audience.

**Failure mode:** A life score disguises subjective judgment as measurement. Refuse composite
scores. Preserve contradictions and celebrate experiences that leave no machine-readable
trace.

**Graduation test:** At least one commitment or automation is consciously released, the
next season becomes clearer, and the person still recognizes the story as their own.

## Connector choreography: powerful combinations

The most valuable use cases often emerge between connectors rather than inside one. These
combinations illustrate the pattern:

| Choreography | What becomes possible | Essential boundary |
| --- | --- | --- |
| Gmail + Calendar + Docs | An email commitment becomes a proposed calendar hold and a living brief | Do not send or move events without approval |
| Zoom/Granola + Jira + Slack | Decisions become owned work and a concise team recap | Confirm owners and visibility before posting |
| Telegram Bot + Family brain + Tasks | Voice notes and group messages become shared logistics | Keep private memories out of the group |
| Oura/Whoop + Strava + Calendar + shell | Recovery, training, and travel become a reviewable personal experiment | Correlation is not diagnosis or causation |
| GitHub + Jira + Impact brain + Docs | Delivery evidence becomes a sourced professional narrative | Credit collaborators and protect employer data |
| HubSpot + Gmail + Stripe | Relationship change, communication, and revenue context reveal customer risk | Human review before outreach or commercial action |
| Plaid + Sheets + shell | Messy transactions become a reproducible monthly exception report | Read-only by default; never expose credentials |
| Luma + Forms + Maps + Telegram | Registration, preferences, routes, and live coordination become one event workflow | Collect only necessary attendee data |
| Canva/Figma + Docs + social connectors | A core idea becomes reviewed channel-specific creative | Publishing remains explicit and audience-aware |
| Calendar + Maps + Uber/Grab | A changed meeting produces realistic travel alternatives | User approves bookings and spending |

The architecture is simple: **one canonical source, one transformation layer, one reviewed
artifact, one explicit action boundary**. When two systems can both edit the same truth,
choose which one wins before automating synchronization.

## A maturity model for personal agents

Do not jump from a clever prompt to autonomous action. Earn each level:

### Level 0: conversational

The agent answers a question from context provided in the moment. Nothing persists and
nothing happens elsewhere.

### Level 1: remembered

The agent uses a scoped Living Brain with reviewed sources and clear retention boundaries.

### Level 2: connected

The agent reads from the minimum set of integrations and cites the evidence behind its
answer. It can create drafts but not consequential actions.

### Level 3: proactive

One narrow, measurable trigger produces a bundled brief or decision request. Quiet hours,
materiality thresholds, correction, and mute controls are defined.

### Level 4: approval-gated action

The agent prepares an action in the correct system and waits for explicit approval. The
approval screen makes destination, audience, cost, and reversibility clear.

### Level 5: bounded autonomy

Only low-risk, reversible, well-observed actions graduate here. The workflow has a budget,
scope, audit trail, failure behavior, and automatic stop condition. High-consequence actions
remain approval-gated regardless of maturity.

Progress is not mandatory. A Level 2 workflow that reliably saves an hour and preserves
agency may be better than a Level 5 workflow that demands supervision.

## The Life OS scorecard—without a life score

Evaluate each play independently. Never collapse these questions into a single number:

- **Attention:** Did it recover attention or create another inbox?
- **Agency:** Did it clarify a choice or quietly make the choice for me?
- **Truth:** Can I inspect sources, assumptions, transformations, and uncertainty?
- **Care:** Did it distribute coordination fairly and respect the people represented?
- **Privacy:** Was every memory and connector permission necessary?
- **Resilience:** Can it work manually, degrade gracefully, and recover from bad data?
- **Learning:** Did the outcome improve the next decision, or did the brain merely grow?
- **Aliveness:** Did it create more room for presence, curiosity, rest, and meaningful work?

The ambition of life-maxxing is not to become a perfectly managed person. It is to build a
small, humane intelligence infrastructure that carries what machines are good at—recall,
reconciliation, computation, and vigilance—so people can spend more of themselves on
judgment, courage, imagination, and care.

## The four-week launch plan

### Week 1: choose and bound

Pick one play and define its brain, agent role, allowed sources, forbidden data, approval
points, and one useful output. Capture a baseline such as weekly planning time or missed
commitments.

### Week 2: run it manually

Ask the agent to produce the output on demand. Correct its assumptions and sources. Save
only the preferences that should genuinely persist.

### Week 3: make one step proactive

Schedule a predictable brief or define one material trigger. Set quiet hours and provide a
simple way to mute, snooze, or correct the workflow.

### Week 4: keep, change, or delete

Review whether the play recovered attention, improved a decision, strengthened a
relationship, or supported energy. Keep it only if the benefit exceeds the new
notification and maintenance cost.

## Guardrails for a life you still own

- **Approval:** Sending, publishing, paying, booking, deleting, applying, and changing
  production systems require explicit approval.
- **Privacy:** Use separate brains, least-privilege access, and minimum-necessary handoffs.
- **Evidence:** Ask agents to distinguish source facts, calculations, interpretations, and
  suggestions.
- **Health and money:** Treat outputs as organizational support, not professional medical,
  mental-health, legal, or financial advice.
- **People:** Never infer sensitive traits, manipulate behavior, or expose private memories
  in a shared chat.
- **Attention:** Bundle routine updates, honor quiet hours, and make every workflow easy to
  pause or delete.
- **Enough:** Preserve unoptimized time. The system serves the life; the life does not
  serve the system.
