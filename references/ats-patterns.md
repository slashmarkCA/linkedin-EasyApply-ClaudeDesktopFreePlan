# Known ATS Patterns (Easy Apply / embedded flows)

Notes accumulated from real runs. Add to this file as new platforms/quirks appear - don't let SKILL.md itself grow long with this level of detail.

## LinkedIn native ("applicantTrackingSystemName=LinkedIn")
- Multi-page modal (varies 3-4 pages): contact info -> resume -> screening questions -> review.
- Resume radio options show whatever's already saved to the LinkedIn profile (e.g. "Resume - Generic - Your Name.pdf" and "Resume - Your Name.pdf") - no local file path needed, just click the right radio.
- LinkedIn sometimes pre-selects the resume used in the MOST RECENT prior application, not necessarily the right one for this posting - always verify the radio selection, don't assume.
- "Follow this company" checkbox is usually checked by default on the final review page - uncheck per config.
- After submit, a "Next best action" / identity verification popup often appears - dismiss it ("Not Now" or "Dismiss"), never engage.
- `browser_find` searches page content regardless of whether a dialog is visually on top - if a dialog appears to have "vanished" after an action, take a full snapshot before assuming it closed. It may still be open with the search just matching background content.

## Zoho Recruit (embedded, "applicantTrackingSystemName=Zoho")
- Longer flow (7 pages seen), does NOT reuse LinkedIn-saved resumes - requires a fresh file upload via file picker, using the `filePath` of the `resumes` entry in config that matches this posting (see the resume-selection step in `SKILL.md`).
- Location fields are typeahead - after filling text, a dropdown suggestion must be explicitly clicked to commit the value, not just typed and left.

## Ceipal (embedded, "applicantTrackingSystemName=Ceipal")
- Shorter flow (~5 pages), resume selection reuses LinkedIn-saved resumes like native Easy Apply (no file path needed here despite being an external ATS).
- Education fields can be blank on the LinkedIn profile even when everything else is populated - watch specifically for a required "School" field with no value; ask the user rather than guessing.
- Clicking "Review" can trigger inline field validation errors that look like a new step, not a real page transition - read the error state before assuming something broke.

## Bullhorn (embedded, "applicantTrackingSystemName=Bullhorn")
- Simplest flow seen (3 pages), no "Follow company" checkbox on review.
- Resume selection has correctly defaulted to the last one used in the session so far - still verify, don't assume this holds forever.

## General
- A posting's own stated "Submission Deadline" can be in the past even while LinkedIn still lists it as open - check for explicit deadline language in the description before investing time in a multi-page apply flow.
- Government/OPS contract postings and healthcare-domain postings tend to have the longest, most detail-heavy forms - budget more tool calls for these.
