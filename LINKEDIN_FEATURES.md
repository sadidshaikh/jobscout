# LinkedIn features: how to use them

There are two parts:

1. **Apply MCP tools** (`tools/apply-mcp/`). You use them by asking Claude Code in plain words. Claude runs the tools and shows you the results.
2. **The watcher** (`tools/watch/`). It runs on GitHub Actions on its own and sends job alerts to your phone.

---

## Setup

### MCP server
The server is registered in `.mcp.json` and builds itself on start, so you only need `pnpm install`.

If `apply` shows as disconnected in Claude Code, run `/mcp` and reconnect it, or restart Claude Code.

### Chrome for the logged-in tools
Some tools act as you on LinkedIn, or fill forms in a browser. For those, start Chrome with remote debugging:

```sh
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --remote-debugging-port=9222 --user-data-dir=$HOME/.jobscout/chrome-profile
```

The first time, log into LinkedIn in that window. The login is saved in that profile, so later you only run the command again. It's separate from your normal Chrome.

| Tool | Needs Chrome? |
|---|---|
| `jobs_search`, `job_detail` | No |
| `posts_search`, `form_read`, `form_fill`, `form_submit`, `page_read` | Yes |

---

## Part 1: The 7 MCP tools

### 1. `jobs_search`: find LinkedIn job listings, ranked for you
It searches LinkedIn's public job listings and scores each job from 0 to 100 against `src/lib/profile.ts`. The score looks at:
- **Stack:** Java, Kotlin, Spring Boot
- **Level**
- **Experience asked:** the profile has 3 years, and roles asking up to about 3 get a bonus
- **Blockers:** off-stack roles and very senior roles score lower

| Option | Meaning | Default |
|---|---|---|
| `keywords` | search terms (required) | — |
| `location` | location | India |
| `experience` | internship / entry / associate / mid-senior / director / executive | any |
| `workplace` | remote / hybrid / onsite | any |
| `postedWithinDays` | 1–30 | 7 |
| `pages` | 10 jobs per page, up to 5 | 2 |
| `limit` | max results (≤50) | 20 |
| `minScore` | hide jobs scoring below this | 0 |

**Example prompts:**
- "Search Spring Boot jobs in Pune posted in the last 2 days"
- "Find remote Java backend jobs, mid-senior, minScore 65"
- "Kotlin backend jobs in Bangalore this week, top 10"

### 2. `job_detail`: full details of one job
It shows the requirements, criteria and number of applicants. It also shows **where the Apply button goes**, with the link:
- `easy-apply`
- `google-form`
- `ats`: a company's job application site, such as Greenhouse or Lever
- `external`: another site

Options: `jobId` (required) and `maxChars` (description length, default 1800).

**Example prompts:**
- "Show details for job 4012345678"
- "Open the 2nd one from that list"

### 3. `posts_search`: find recruiter "we're hiring" posts
Many jobs, especially at startups and through recruiters, are only posted as LinkedIn posts, like "Hiring Java devs, fill this form / DM me". They never appear as job listings.

**What it returns:** one ranked line per post, with:
- role and company
- location
- years of experience asked
- how to apply: Google Form / company application site / careers page / email / DM
- fit score

**What it drops:** "open to work" posts, posts that aren't about hiring, internships, big experience gaps and locations outside the profile. It reports how many it dropped.

| Option | Meaning | Default |
|---|---|---|
| `roles` | 1–6 short role phrases, e.g. `["java developer", "spring boot"]`, not sentences | required |
| `days` | `1`, `7` or `30` | 7 |
| `sort` | `latest` / `relevance` / `both` | both |
| `perQuery` | posts read per search (3–30) | 12 |
| `textChars` | include this much post text per post | 0 |
| `includeDropped` | also list the posts it dropped | false |
| `includeInternships` | keep internship posts | false |
| `query` | write your own LinkedIn search instead | — |

It uses your logged-in account, so **keep it to a few searches a day** to avoid LinkedIn flagging you.

**Example prompts:**
- "Find hiring posts for java developer, spring boot, kotlin backend from the last 7 days"
- "Hiring posts from today for backend developer, show post text"

### 4. `form_read`: read an application form
It opens a Google Form, a company application site, or any page with a form. It lists every question as a numbered field, already answered from the answer bank. It also returns a form id (e.g. `f1`) and marks the required questions it couldn't answer.

**Example prompts:**
- "Read this form: <url>"
- "Read the apply form for that job"

### 5. `form_fill`: fill the form (does NOT submit)
It fills the fields and attaches your resume. Then it reads back what the page actually shows and flags anything that doesn't match. Answers you give override the bank. For checkboxes, join values with `;`.

Options: `form` (the id from `form_read`), `answers` (field number → value) and `resume` (a resume key from `applicant.json` or an absolute path).

**Example prompts:**
- "Fill it"
- "Fill it but put expected CTC as 20 LPA"

### 6. `form_submit`: submit
It submits the filled form, and only after you explicitly approve that specific application. Look over the filled form in the Chrome window first.

**Example prompt:** "Looks good, submit."

### 7. `page_read`: read any page
It opens a URL, or uses the page already open, and returns the visible text plus any apply or form links. It's useful for careers pages or when an apply link goes somewhere odd.

**Example prompt:** "Read this careers page and find the apply link: <url>"

---

## Typical flow

1. **Find:** "Find hiring posts for java developer, spring boot" and/or "search Spring Boot jobs, last 3 days".
2. **Check:** "Details on #3". Claude tells you the stack, experience asked and how to apply.
3. **Read form:** "Read its form". Claude shows you the pre-filled answers and asks only about the gaps.
4. **Fill:** "Fill it". You check it in Chrome.
5. **Submit:** "Submit". If it's a DM or email route, ask Claude to draft the message instead.

**One limit:** LinkedIn **Easy Apply** popups aren't one of the supported form types. The form tools target Google Forms, company application sites and normal web forms. Apply to Easy Apply jobs yourself on LinkedIn.

---

## Answer bank

The file is `tools/apply-mcp/private/applicant.json`. It's **gitignored**, so it's never pushed. It holds:
- name, email, phone and links
- resume path(s)
- current and expected CTC, and notice period
- years of experience, location, and whether you'll relocate or work from office/hybrid
- degree, college, graduation year, CGPA and skills

When a form asks something new, say "save that to the bank" and future forms will use the answer.

---

## Part 2: The watcher (phone alerts on its own)

- **What it does:** runs every search in `tools/watch/searches.json` over the last 12 hours. The current searches cover Java, Spring Boot, Kotlin, backend, SDE 2, software engineer, remote and Pune.
- **Alerts:** it sends a phone notification through **ntfy** for any job scoring **70 or more**. Change the cutoff with the `WATCH_THRESHOLD` setting.
- **Where it runs:** GitHub Actions (`.github/workflows/watch.yml`), every hour. You don't have to keep anything running on your laptop.
- **Run it locally:** `pnpm watch` shows what it would alert on right now.
- **Optional:** `DATABASE_URL` stores the alerts so the app's `/alerts` page can show them.

### Turn it on
1. Install the **ntfy** app on your phone.
2. Subscribe to a topic name that's hard to guess, e.g. `yourname-jobs-x7k2`.
3. Add it as a repo secret: `gh secret set NTFY_TOPIC --repo <owner>/jobscout`.
4. Trigger a test run: `gh workflow run watch.yml --repo <owner>/jobscout`.
5. Optional: GitHub often skips hourly scheduled runs. A free cron-job.org job that calls GitHub's API to start the workflow can run it every 15 minutes.

### Tuning
- Searches: edit `tools/watch/searches.json`.
- Scoring, stack and experience: edit `src/lib/profile.ts`.
