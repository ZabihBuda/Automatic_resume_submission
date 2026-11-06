# Job Auto-Apply: an end-to-end job application agent

A Python tool I built and use every day in my own job search. It finds new data science and
machine learning openings, ranks them against my resume, fills in and submits the real
application forms in a browser, and keeps a log of every result. It runs by itself once a
day on my Mac and applies to about 40 roles a day across the US and Canada.

> The source code is in a private repository because it holds my personal application
> data. I'm happy to walk through the code or give access on request.

## What it does

- **Finds jobs** from the public Greenhouse, Lever and Ashby job board APIs of about 70 tech
  companies. These are open, read-only feeds, so there's no scraping and no login.
- **Filters and ranks** them: data science / ML / NLP / LLM titles only, seniority and
  location rules, work-authorization checks, then a score based on how many of my resume
  skills each posting mentions, with a boost for preferred locations.
- **Never applies twice.** Every posting ever seen is stored in SQLite, reposts with the same
  title are recognised, and there's a per-company daily cap.
- **Fills in the real forms** with Playwright: contact details, resume upload, employment and
  education, React dropdowns, searchable location pickers, checkboxes and radio groups.
- **Answers screening questions** from a rules file first, then sends any free-text questions
  to the Claude API in one request per application, grounded only in my resume and the job
  description, and told to report "not confident" rather than guess.
- **Submits and verifies.** It clicks Submit, waits for the confirmation page, and saves a
  screenshot of every form.
- **Handles email verification codes** by reading them from Gmail over read-only IMAP
  within seconds of submitting.
- **Knows when to stop.** Forms with a CAPTCHA, or a required question it can't answer, are
  skipped, never guessed. The open questions go into a file; once I answer them, every
  later form that asks the same question reuses my answer.
- **Runs on a schedule** through a macOS LaunchAgent: once a day at login or wake, with a
  desktop notification summarising what was submitted.

## How it works

```
fetch ~70 job boards (Greenhouse / Lever / Ashby APIs)
   │
   ▼
filter by title, seniority, location, authorization ──► score against resume ──► dedupe (SQLite)
   │
   ▼
daily queue: ~20 US + ~20 Canada, newest postings first, max 3 per company
   │
   ▼
for each job (Playwright):
   open form → fill profile + resume → answer questions (rules → Claude API)
   → CAPTCHA?  skip and report
   → Submit → email code? read it from Gmail → confirmation page?
   │
   ▼
applied / needs_input / captcha / closed  → applications.db + screenshot + daily notification
```

## Engineering highlights

- **Messy real-world forms.** Greenhouse alone serves several form generations, including
  React-based dropdowns and location fields that only accept a value picked from a live
  search list. The autofill layer finds each question by its label, works out the field
  type, and picks the closest option to the intended answer.
- **Cost-aware LLM use.** Rules answer most questions for free. Only what's left goes to
  Claude (Haiku, low effort) in a single batched request, so a day's run costs about three
  cents. Without an API key it falls back to templates and still runs.
- **Safe by design.** It only uses sources whose terms allow automated access (no LinkedIn or
  Indeed), never bypasses CAPTCHAs, only reads the inbox, and keeps every credential in a
  git-ignored local config.
- **Resilient runs.** Closed postings are detected and dropped, failed jobs get one retry,
  a browser crash doesn't stop the run, and a missed day is picked up at the next hourly
  check.
- **Tested.** About 45 pytest tests, including Playwright tests against saved copies of real
  application forms (React dropdowns, CAPTCHA pages, email-code prompts, confirmation pages)
  and a mocked IMAP server.

## Tech stack

Python · Playwright · SQLite · Claude API · Greenhouse, Lever and Ashby APIs · IMAP ·
macOS launchd · pytest

## Project structure

```
autoapply/
  sources.py   job board API clients (Greenhouse, Lever, Ashby)
  filters.py   title, location and authorization filters, resume-based scoring
  db.py        SQLite store: jobs, applications, dedupe, retry state
  autofill.py  Playwright form filling, question answering, submit and verify
  gmail.py     read-only IMAP polling for emailed security codes
  cli.py       commands: run, fetch, review, retry, log, check-key, check-gmail
scripts/
  schedule_mac.sh   install / remove the daily LaunchAgent
tests/         pytest suite with saved HTML copies of real forms
```

## About me

I'm Zabihullah Zabih, a data scientist working in ML, NLP and LLM applications. I built this
to run my own job search the way I'd build any production pipeline: reliable sources,
clear ranking, measurable outcomes and a full audit log.
