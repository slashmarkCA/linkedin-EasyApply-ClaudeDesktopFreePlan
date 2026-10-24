# LinkedIn Lightweight Easy Apply Agent (Claude Skill)

A config-driven Claude Skill that searches LinkedIn for job postings and submits Easy Apply applications automatically, using your own filter/preference rules. Built to run on **Claude Desktop's free plan** using open MCP servers and Playwright - no Claude Code or Pro subscription required.

Businesses using Linkedin are wasting job hunter time and dignity, so take back what you can.

## Deprecated Project
This project will not be maintained as the way AI retains retains page DOM with every page it loaded became unmanagable and because useless at scale and manages to get 6 applications completed at best while hunting through posting level pages for a plethora of keywords.
Lesson learned, but fear not, the [roleRecon](https://github.com/slashmarkCA/roleRecon) project is switching to Python scraping first, and will expand beyond Easy Apply focus and more, which was a logical feature addition for this project anyway.

## Why it's built this way

This project's explicit goal was to stay on Claude Desktop's free plan.

If you're using Claude Code or Claude Cowork instead, you likely don't need any of this except the linkedin_config_example.json customization logic - those products have native browser/agent tooling built in, the MCP + Playwright setup used here becomes unnecessary. 


- **Browser control** comes from the open-source [Playwright MCP server](https://github.com/microsoft/playwright-mcp), not Claude Code's built-in browser tooling.
- **File read/write** comes from the [filesystem MCP server](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem), sandboxed to specific folders you choose.
- **The "skill" concept.** (a `SKILL.md` file the agent reads before acting) is Anthropic's own pattern, available on the free plan.
- **Why is the filesystem MCP necessary?** No cowork availability notwithstanding, not everyone is comfortable editing .json and markdown files - you can tell Claude to make changes for you with the less painful and infrequent tradeoff to have to upload a new skill file from time to time. 
- **Why point this to 2 or more local resumes if LinkedIn already has them?** Some 3rd party ATS' wrapped in LinkedIn's site make you upload a resume from your hard drive, even though you have some inside your linkedin profile.
- **Why no node.js, Windows Credential Manager -> HashiCorp Vault, cloudflare workers, followed by a Redis yadda yadda?**  Exactly.  This project is not an unneccessary attempt to show off vibecoder madSkillz to recruiters for something simple.
- **Why make me watch AI working in a browser that slows us down when it should "just do it"?** Because watching what AI encounters the first few times you run it lets you think about new ways to train Claude to modify your preferences when you undoubtedly will encounter "Hmm, I hadn't considered that" moments.  "Human in the middle" = Good.  Tell Claude to turn it off (say "Go Headless") whenever you want and just answer its questions in the desktop chat. 
- **This idea sucks - I keep running out of tokens.** Yeah, it happens.  If you're comfy blowing $25-$100/week applying to companies you'll never hear from with a paid account, then knock yourself out and buy me a coffee for prototyping a skill for free that you can bring into Claude Pro.

## What this actually does

1. Reads+Writes your personal settings in `linkedin_config.json`.
2. Searches LinkedIn for jobs matching your criteria with a bias to search first for postings that have gone up in the last 36000 seconds (10 hours) because, face it, anything online for more than a day and you stand zero chance when hundreds of other people have applied ahead of you - as if beating Applicant Tracking Systems wasn't bad enough!
3. Filters out postings you've told it to exclude (bad titles, language requirements, disliked platforms, etc.) and ranks the rest by your stated preferences.
4. Shows you the shortlist for review.
5. For postings you approve, fills out the Easy Apply form (or an embedded external ATS like Zoho/Ceipal/Bullhorn) using your saved data, and submits.
6. Logs every application to a local CSV, and optionally to a Google Sheet.

**What it does NOT do (by design, for now)**: apply to postings that redirect fully off LinkedIn to a company's own career site (Workday, Ultipro, etc).  Nor does it attempt to help you rewrite your resume - this project is to store keyword refinements and synonym awareness to locate postings ("Clerk" can mean a cashier, a receptionist, a law office worker, and many different things to the human posters and also AI interpreting what you're looking for).  If you need help with your resume, from experience, I highly recommend NOT using AI. Not at all.  They're garbage.

---

<details>
<summary><strong>Setup</strong></summary>

### Prerequisites

1. **Node.js** (v18+) - required to run the MCP servers via `npx`. [Download here](https://nodejs.org/). Verify with:
   ```
   node -v
   ```
2. **Claude Desktop** - free plan is sufficient.

### 1. Install the MCP servers

Open (or create) your Claude Desktop config file:
- Mac: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

Add the `playwright` and `filesystem` servers. The `filesystem` server's `args` list is a sandbox - only the folders you list are ever readable/writable by Claude. At minimum you need the folder you'll clone this repo into:

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"]
    },
    "filesystem": {
      "command": "npx",
      "args": [
        "-y", "@modelcontextprotocol/server-filesystem",
        "C:\\path\\to\\where\\you\\cloned\\this\\repo",
        "C:\\path\\to\\your\\resume\\folder"
      ]
    }
  }
}
```

Fully quit and restart Claude Desktop after saving.

### 2. Clone this repo

Clone it somewhere inside a folder your `filesystem` MCP server can see (per the config above). A common convention: `~/.claude/skills/linkedin-inapply-agent/`.

### 3. Upload your resume(s) to LinkedIn first

Before running anything, upload at least one resume to your LinkedIn profile (via any job posting's Easy Apply flow, or your profile's resume/documents section). The skill selects from resumes already saved to your LinkedIn account - if none exist yet, it will stop and ask you to add one rather than guessing or failing silently.

### 4. Set up your config

```
cp linkedin_config_example.json linkedin_config.json
```

Fill in your own data - personal info, resume file paths, search keywords, filters, salary logic, etc. Every field has an inline note explaining what it's for. **This file is gitignored and will never be committed** - it contains your personal data.

### 5. (Optional) Set up logging

See the Logging section below - it's entirely optional and the skill works fine with zero setup here.

### 6. Install the skill into Claude Desktop

In a chat with Claude Desktop, ask Claude to package this folder as a `.skill` file and present it to you - it'll walk you through zipping `SKILL.md` and `references/` and give you a file card with a **"Save skill"** button. Click it. That installs the skill into your Claude account.

I recommend creating a Project to organize yourself.

**Important**: Claude Desktop's installed skill and this local folder are two separate things. If you edit `SKILL.md` after installing, you need to re-package and click "Save skill" again for the change to take effect - Claude Desktop doesn't watch this folder live. (If you later add Claude Code, it *does* read `~/.claude/skills/` live, no packaging needed - one more reason this repo defaults to that folder location.)

### 7. Try it

In a new chat, type `/`, pick this skill from the list, and add a job title or keyword to search for (or a job ID to apply to). With no keyword, it starts with the first one in your config and offers the next after each shortlist.

## Session Pollution and Context Window Bloat
If you run a massive job sweep that opens, scrapes, logs, and processes  many many job pages in a single long chat session, the LLM's context window fills up with DOM text dumps, leading to dropped filters or forgotten rules near the end of the batch.
So - treat chat sessions as ephemeral. Do a few applications, log them to CSV/gSheets, close the chat, and spin up a fresh session where it reloads everything fresh, so you can just continue on without explaining what you want to do.

</details>

<details>
<summary><strong>Configuring linkedin_config.json</strong></summary>

Each top-level block does one job:

- **`search`** - your job title keywords (an ordered list, searched one at a time), location, radius, and how recent a posting must be.
- **`workflowPreferences`** - your approval trigger words (`approvalCommands`), whether to follow companies after applying, and which button dismisses LinkedIn's post-submit popups (Like no, for the 800th time, I do not want to Verify myself or buy LinkedIn Premium!).
- **`filters`** - everything to exclude a posting (bad title keywords, disqualifying body-text terms, compensation types to skip, or multi-lingual must haves), plus `preferredKeywords`, which only nudges ranking rather than excluding anything.
- **`personal`** - your contact info and LinkedIn URL, used to fill contact fields.
- **`resumes`** - your resume variants. Each has a `label` (any name you want), a `filePath`, and a plain-language `useWhen` description - the agent matches a posting's content against these descriptions to decide which resume fits - the same logic is used when it decides which resumes are uploaded to LinkedIn to choose with those radio buttons, and also if a 3rd party form forces you to upload a pdf off your hard drive. Default is spots for 2, but add as many as you actually have.  
Think of it this way - you are applying to McDonalds and NVidia CEO jobs at the same time - you probably need 2 different resumes.
- **`workAuthorization`** - your work authorization, sponsorship, and relocation answers for screening questions.
- **`workArrangement`** - three switches, `acceptRemote`, `acceptHybrid` and `acceptOnsite`. A posting whose arrangement is switched off is excluded, and the same switches answer "are you comfortable working onsite/hybrid/remote?" questions. Postings that don't state an arrangement are flagged to you rather than excluded.
- **`experienceSummary`** - `defaultExperienceYears` (your main field in a few words, plus your total years in it, used for general questions about that field), `skillYears` (a skill-to-years map used to answer "How many years of experience do you have with X?" questions; `0` means none, and `null` or a missing skill makes the agent ask you instead of guessing), `keySkills` (a secondary signal for picking a resume), `levelMismatchThresholdYears` (how many years a posting's stated requirement must exceed yours before a seniority mismatch gets flagged), and a `background` blurb - note that `background` isn't typed into any literal form field, it informs tone/fit on open-ended questions like "why are you interested in this role."
- **`education`** - school, degree, and dates, for forms that ask.
- **`screeningQuestionDefaults`** - default answers to common screening questions (salary logic, why you're interested in the role, degree status, etc.) - anything not covered here, the agent asks you rather than guessing. Notice period and start date aren't stored: the agent answers "Immediate" and tomorrow's date.
- **`coverLetterBlurb`** - a reusable short blurb the agent can adapt if a posting asks for a cover letter.
- **`logging`** - on/off switch for the CSV log (Google Sheets logging is controlled separately, via `.env` - see the Logging section below).

</details>

<details>
<summary><strong>Logging</strong></summary>

Every submitted application gets logged. Two options, and you can use either, both, or neither:

### CSV (default, zero setup)

Every application appends one row to `applications_log.csv` in this folder (created automatically on first use). Columns: `Application DateTime, Job Title, Company, LI_URL, Resume Used, Notes`.

**To turn this off**: set `"csvEnabled": false` in the `logging` block of your `linkedin_config.json`.

### Google Sheets (optional)

If you want a shared/cloud copy instead of (or in addition to) the CSV, you can wire up a Google Sheet via Apps Script - no Google Cloud project or API key needed, just a script deployed to your own sheet.  Yes, you could import the .csv into google sheets and skip this, but the convenience is nice.

**To turn this on:**

1. Create a Google Sheet with headers in row 1: `Application DateTime | Job Title | Company | LI_URL | Resume Used | Notes`
2. In the sheet, go to **Extensions → Apps Script**, and replace the default code with:

   ```javascript
   function doPost(e) {
     var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
     var data = JSON.parse(e.postData.contents);
     sheet.appendRow(data.row);
     return ContentService.createTextOutput(JSON.stringify({"status": "success"}))
       .setMimeType(ContentService.MimeType.JSON);
   }

   function doGet(e) {
     return ContentService.createTextOutput(
       JSON.stringify({ status: "alive", message: "Web App is active. Use POST to send data." })
     ).setMimeType(ContentService.MimeType.JSON);
   }
   ```
   (The `doGet` isn't used by the skill - it just lets you sanity-check the URL is live by visiting it in a browser.)

3. **Deploy → New deployment → Web app.** Set **"Execute as"** to **Me**, and **"Who has access"** to **Anyone**. Deploy, and copy the URL it gives you (ends in `/exec`).
4. `cp .env.example .env`, and paste that URL as `GOOGLE_SHEETS_POST_URL`.

**Deployment gotchas**:
- Editing the script code does **not** update a live deployment. After any code change, go back to **Deploy → Manage deployments**, edit your deployment, and select **"New version"** before redeploying - otherwise the old code keeps running at the same URL.
- If you ever get a `403` with an empty response body, double check "Who has access" is genuinely set to "Anyone," and that you redeployed as a new version after any settings change.
- **The response from this endpoint cannot be trusted.** This is a quirk of Apps Script Web Apps: a call that successfully appends your row can still return an unrelated response body (it returns the `doGet` message, and even a Google Drive "file not found" error page) on a fully successful write. The skill is written to never parse or rely on this response - it fires the request and moves on, treating the CSV as the only confirmed record. Ignore it if you inspect Dev Tools' network traffic and see what looks like an error; check the sheet directly instead.
- Treat this URL like a password once deployed - anyone who has it can write rows to your sheet. It lives in `.env` and gitignored for that reason.

</details>

<details>
<summary><strong>Passwords &amp; credentials</strong></summary>

**This project does not store, request, or manage any login passwords**, by design.

LinkedIn access works through Playwright's persistent browser profile - you log in manually, once, in the browser window Playwright opens, and that session stays authenticated across runs. No password is ever typed by the agent, read from a file, or seen by Claude.

If you venture out of Easy Apply, the recommended approach is to let the browser's own built-in password manager generate and remember a unique password locally, rather than storing one anywhere in this repo or its config for security.

</details>

<details>
<summary><strong>Known limitations</strong></summary>

- **External-site applications are out of scope.** Postings that redirect fully off LinkedIn to a company's own career portal (Workday, Ultipro, iCIMS, etc.) are intentionally not automated. LinkedIn-embedded flows (native Easy Apply, and embedded external ATS like Zoho Recruit, Ceipal, Bullhorn - which render *inside* LinkedIn's own page) are supported; a genuine full redirect off the site is not, because there's no shared form convention to template against across thousands of different employer portals.
- **Indeed is not supported.** Tried and abandoned - its search relevance proved too literal to reliably find relevant postings for this project, even under patient manual testing.
- **"Personas" switching incomplete** The config is designed with the ability to have 2 or more resumes - E.g. apply to a Bricklayer job or CEO of Nvidia job, *but it needs more work*. \
Sometimes custom ATS forces you to input a plain text cover letter but there is only one available in the config. Also, the salary range only gives one value too, but asking for CEO's hourly rate in a Bricklayer application is nonsensical.

</details>

---

## License

Released under the [0BSD license](LICENSE).

This is a personal automation tool, shared as-is for others building something similar or I prefer bringing in collaborators - especially when we take on applying to external sites that require you to sign up for an account one by one, and their designs are all wildly different and horrifically coded.

## Risks and terms 

**Automating LinkedIn is against LinkedIn's rules.** LinkedIn's User Agreement does not permit third-party software, including bots and browser plug-ins, that automates activity on its site. Using this tool can lead to LinkedIn restricting or closing your account, so do not use it on an account you can't afford to lose.

- **It fills in real applications for you.** Applications can't be unsent, and every answer comes from your own `linkedin_config.json`. Check it, and review what the agent shows you before approving.