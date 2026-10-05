---
name: linkedin-inapply-agent
description: Search LinkedIn for job postings and submit Easy Apply applications using the data in linkedin_config.json. Invoke with a job title or keyword to search, or a job ID to apply to.
---

# LinkedIn Easy Apply Agent

Automates searching LinkedIn and submitting Easy Apply applications on the user's behalf, using Playwright MCP for browser control and a personal `linkedin_config.json` config file for all variable data, handwritten by the user, or by the AI agent by receiving directions and values from the user through chat questioning.

## Session start

At the start of a new chat before executing any tasks:
1. Read linkedin_config.json for user and search preferences.
2. Read HANDOFF.md for open to-do items and ephemeral prior chat context being carried forward to a new chat.  If it does not exist, create it after asking the user's permission, to store ideas to modify this skill or their linkedin_config.json key/value pairs at a later date.

`linkedin_config.json` is the user-specific data (personal info, filters, resumes, preferences). 
`linkedin_config.json` is gitignored and never committed because it contains personally identifiable information.
`linkedin_config_example.json` is the blank schema baseline to be replicated as `linkedin_config.json` in the local filesystem if it is missing and used as the operational file.  
Then proceed to Step 1.


## Step 1: Search
Inspect `environment.runPlaywrightHeadless` to decide how to launch the browser mode.

Search ONE keyword at a time, never several in one query (LinkedIn's search box takes a single string).
- If the user's message names a job title or keywords, search exactly that.
- Otherwise start with the first entry in `search.keywords`.
- After presenting the shortlist, say how many postings survived the filters and offer the next entry in `search.keywords`. If the user says yes, search it and repeat.

Apply `search.filterToLinkedInApplyOnly` (as the 'LinkedIn Apply' filter chip / `f_AL=true`) to every search when true.

Always apply `search.location`, `search.distance_km`, `search.postedWithinSeconds` (as `f_TPR=r<seconds>`).

Cap pagination at ~75 results / 3 pages per run unless told to go deeper.

Title matching against LinkedIn's own search relevance is fuzzy - always eyeball results even when the title string doesn't obviously match a keyword, since a good fit is often posted under an unrelated title.

## Step 2: Filter and rank

All matching in this step (titles, employer names, description terms) is case-insensitive substring matching, not exact/whole-word matching.

For every result title, check against `filters.excludedTitleKeywords` and `filters.excludedEmployers` - drop silently, don't show these to the user.

For postings that survive the title filter (the shortlist), open each one and check `filters.descriptionExclusions` (language requirements, technical requirements, platform exclusions). Hard exclude any match. One standing exception: `platformExclusions` listing Guidewire is scoped specifically to the Claims/ClaimCenter module - a posting mentioning Guidewire PolicyCenter or ContactManager is NOT excluded on that basis alone.

Check `filters.excludedCompensationTypes`, `filters.excludedEmploymentTypes`, and `filters.minSalaryCAD` (if set, exclude postings stating less) - hard exclude on any of these regardless of title match.

Apply `workArrangement`: classify each posting as Remote, Hybrid, or On-site from LinkedIn's workplace-type tag on the posting. If there is no tag, read the description. If it is still unclear, do NOT exclude it - flag it to the user as "arrangement unclear" when presenting. Hard exclude a posting when the flag for its arrangement (`acceptRemote`, `acceptHybrid`, `acceptOnsite`) is `false`.

Weight remaining results toward matches in `filters.preferredKeywords` when presenting or ranking - this is a tiebreaker signal, not a filter. Mention the specific match when presenting a result that scored higher because of one.

Apply `experienceSummary.levelMismatchThresholdYears` - do not flag "Lead"/"Senior"/"Manager" titles as mismatches on their own; only flag a level concern when the posting's stated minimum years clearly exceeds that threshold, or when it's a different discipline entirely (e.g. manufacturing/food QC vs software QA) - not just because the title sounds senior.

Present the surviving list to the user for review before applying to anything, unless they've already said an approval word (see `workflowPreferences.approvalCommands`) on specific postings.

## Step 3: Apply

For each approved posting:
1. Click the Apply/Easy Apply button, note which ATS it routes to (LinkedIn native, or an embedded external one - Zoho, Ceipal, Bullhorn seen so far). See `references/ats-patterns.md` for known quirks per platform.
2. Select the correct resume: compare the posting's content against each entry in the `resumes` array's `useWhen` field (and against `experienceSummary.keySkills` as a secondary signal), and pick the best-matching entry's `label`. If no entry confidently matches, ask the user rather than guessing. This same matching decision applies to BOTH resume-selection mechanisms, not just one or the other:
   - If the ATS shows saved resume options as radio buttons (native LinkedIn Easy Apply, and some embedded ATS like Ceipal) - select whichever radio corresponds to the chosen entry's resume (matched by filename), don't assume the default/pre-selected radio is correct. If ZERO saved resume options are shown at all, STOP - do not proceed with the application. Tell the user they need to upload at least one resume to their LinkedIn profile first (via the Easy Apply upload option, or their profile's resume/documents section), then retry once done. Never treat an empty resume list as something to work around silently.
   - If the ATS requires a fresh file upload (e.g. Zoho Recruit) - use the chosen entry's `filePath`.
3. Fill any blank required field from `personal`, `workAuthorization`, `workArrangement`, `experienceSummary`, `education`, `screeningQuestionDefaults` in config. See "Guard Rails" below for what to do if a field has no config value. Exception: `experienceSummary.background` is NOT a literal field value to copy anywhere - no LinkedIn form has a field called "background." Use it instead to inform how you describe fit or tone when a posting asks an open-ended question (e.g. "Why are you interested in this role?", "Tell us about yourself"), drawing on its content rather than pasting it verbatim.
   - **Years-of-experience questions** ("How many years of experience do you have with X?") are answered only from `experienceSummary`, in this order:
     1. If `skillYears` has a key matching the skill (case-insensitive, ignoring parentheticals and qualifiers such as "(Programming Language)" or "(AI)"), enter that number exactly. `0` is a real answer.
     2. Otherwise, if the question is about the user's core field in general (it matches `defaultExperienceYears.field`) and `defaultExperienceYears.years` is not `null`, enter that number.
     3. Otherwise STOP and ask the user. A `null` value or a skill with no entry counts as no answer. Never apply `defaultExperienceYears.years` to skills in `keySkills` one by one, and never estimate.
     - Enter the config number even when the posting asks for more; never round up to meet a stated requirement.
     - Ignore any config key that starts with `_`; those are notes for humans.
     - When the user supplies a new skill's years mid-session, offer to add it to `skillYears` (show the diff first, per the Guard Rails).
   - **Work-arrangement questions** ("Are you comfortable working onsite / hybrid / remote?"): answer Yes if the matching flag in `workArrangement` is `true`, No if it is `false`.
   - **Exclusions revealed mid-form:** if a screening question or page inside the application shows a requirement matching `filters.descriptionExclusions` (for example, a required-years-of-Java question when Java is excluded), stop that application. Close the dialog without submitting (discard the draft if prompted), do not log it as applied, tell the user which exclusion triggered, and move on to the next approved posting. Having a `0` for that skill in `skillYears` does not override this.
4. For salary/rate questions, apply `screeningQuestionDefaults.salaryExpectationPolicy` exactly as written - don't add extra parsing logic.
   - Notice-period and start-date questions are not stored in config. For a notice period, answer "Immediate" (enter 0 if the field is numeric). For an available-start-date field, enter tomorrow's date, taken from the browser's local clock via `browser_evaluate` (never from memory), in whatever format the field expects. Never enter a date further out.
5. If a cover letter field is present, use `coverLetterBlurb.template` as a starting point, lightly adapted to the specific posting rather than pasted verbatim.
6. On the final review page, uncheck "Follow this company" per `workflowPreferences.followCompanyOnApply`.
7. If the user already used one of `workflowPreferences.approvalCommands` for this posting after reviewing it, submit immediately. Otherwise stop and show them the filled review page first.
8. Dismiss any post-submit "verify your identity" / "next best action" popup per `workflowPreferences.verificationPopups` - click "Not Now," never engage with the verification flow.
9. Log the application immediately after this specific submission succeeds - not batched at session end. See "Logging" below.

## Logging

After every successful application submission (one row per application, right after that submit, never batched):

**CSV (always, unless disabled)**: check `logging.csvEnabled` in config - if `false`, skip this entirely. Otherwise append one row to `applications_log.csv` in this folder. If the file doesn't exist yet, create it with a header row first. Never overwrite existing rows - append only. Columns, in order:
`Application DateTime,Job Title,Company,LI_URL,Resume Used,Notes`

- `Application DateTime`: use local time, not UTC/ISO (e.g. via `new Date().toLocaleString()` in a browser_evaluate call, which uses the machine's local timezone). Format example: `9/24/2026, 6:27:35 PM`.
- `Notes`: optional free text for anything worth flagging about this specific application (e.g. "stated years slightly below posting's requirement", "applied via 2nd keyword"). Leave blank if nothing notable.

**Google Sheets (optional, additive)**: check `.env` for `GOOGLE_SHEETS_POST_URL`. If present and non-empty:

Important: must be fired from a separate browser tab, never the LinkedIn tab currently in use. LinkedIn's own Content-Security-Policy blocks outbound `fetch()` calls to any domain not on its allowlist, and `script.google.com` is not on it - this is confirmed, not theoretical, and will fail silently/loudly every time if skipped. Procedure:
1. Open a new tab, navigate it to a neutral real page (e.g. `https://www.google.com` - confirmed working; `about:blank` does NOT work, it lacks a valid origin for the fetch).
2. From that new tab, POST to the URL with:
```js
fetch(url, {
  method: "POST",
  redirect: "follow",
  headers: { "Content-Type": "text/plain;charset=utf-8" },
  body: JSON.stringify({ row: [<same 6 values, same order as the CSV row>] })
})
```
3. Close that tab and return focus to the original LinkedIn tab to continue the session.

Never trust, parse, or gate logic on the response status/body even once correctly fired from a neutral tab - Apps Script's Web App redirect-and-execute flow is confirmed unreliable at the response layer for anonymous requests (it can return an unrelated `doGet` message or even a Drive "file not found" error page on a call that still successfully appended the row). Treat every call as fire-and-forget: send it, do not inspect what comes back, do not retry, do not report success/failure to the user based on it. This is additive to the CSV write, not a replacement - the CSV row is the only reliably-confirmed record. If `GOOGLE_SHEETS_POST_URL` is missing or empty, skip Sheets entirely, no error, no nagging.

## Guard Rails
- **Never modify `SKILL.md` or `linkedin_config.json` autonomously.** If a preference changes or a new edge case is found (a new tool needing a years-of-experience answer, a new bad employer, a new exclusion pattern, anything else durable), present the exact diff to the user and get explicit approval before writing to disk.
- **Never fabricate.** If a required form field has no value in config (or the value is blank or `null`), STOP and ask the user. Do not guess, round, infer, or fill in plausible defaults for biographical or professional details (this includes things like start/end months when config only stores years). Once the user answers, use it for the current application and offer to save it to config.
- No diff needed for appending rows to applications_log.csv, writing HANDOFF.md (with permission), or saving config values the user just dictated in chat.