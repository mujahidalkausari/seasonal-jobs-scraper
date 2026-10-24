# Seasonal Jobs & Worker Housing Scraper — H-2A and H-2B job orders through the Apify API

Get U.S. Department of Labor seasonal job orders into your own code as structured JSON:
employer, job title, worker counts, dates, pay, worksite, and the **worker housing addresses,
units and occupancy** that each H-2A job order lists. The data comes from DOL's
[Seasonal Jobs](https://seasonaljobs.dol.gov) registry, and the scraper runs on
[Apify](https://apify.com), so there's nothing to host.

**▶ Run it on Apify: [Seasonal Jobs & Worker Housing Scraper](https://apify.com/alkausari_mujahid/dol-seasonal-jobs-scraper)**

This repository holds documentation and examples for developers. The scraper itself runs
on the Apify platform.

---

## Get job orders with one HTTP request

This returns the most recent agricultural job orders in Florida in the same response:

```bash
curl -X POST \
  "https://api.apify.com/v2/acts/alkausari_mujahid~dol-seasonal-jobs-scraper/run-sync-get-dataset-items" \
  -H "Authorization: Bearer $APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "location": "Florida", "jobtype": "H-2A", "lookbackDays": 20, "top": 50 }'
```

Your API token is under **Settings → API & Integrations** in [Apify Console](https://console.apify.com).
Add `&format=csv` to the URL for a spreadsheet instead of JSON. The synchronous endpoint
waits up to 300 seconds; for larger runs, start the run and fetch the dataset when it
finishes, as below.

## From Python

```bash
pip install apify-client
```

```python
import os
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

run = client.actor("alkausari_mujahid/dol-seasonal-jobs-scraper").call(run_input={
    "location": "Texas",
    "jobtype": "H-2A",
    "lookbackDays": 60,
    "top": 1000,
})

for job in client.dataset(run["defaultDatasetId"]).iterate_items():
    housing = job.get("Housing") or {}
    print(job["Case Number"], job["Employer"], job.get("Total Workers"), housing.get("City"))
```

## From JavaScript

```bash
npm install apify-client
```

```js
import { ApifyClient } from 'apify-client';

const client = new ApifyClient({ token: process.env.APIFY_TOKEN });

const run = await client.actor('alkausari_mujahid/dol-seasonal-jobs-scraper').call({
    location: 'all',
    jobtype: 'H-2B',
    lookbackDays: 20,
    top: 500,
});

const { items } = await client.dataset(run.defaultDatasetId).listItems();
for (const job of items) console.log(job['Case Number'], job['Employer'], job['Employment State']);
```

---

## Input

| Field | Default | What it does |
|---|---|---|
| `location` | `"Florida"` | A U.S. state, district or territory, by name, or `"all"` for every location. |
| `jobtype` | `"H-2A"` | `"H-2A"` (agricultural), `"H-2B"` (non-agricultural), or `"all"`. |
| `lookbackDays` | `20` | Only job orders DOL accepted in this many days before today, from 1 to 365. |
| `offset` | `0` | Day offset from today for the earliest job begin date. `-7` also includes jobs that began up to a week ago. |
| `top` | `10` | Maximum job orders to return, up to 10,000. |
| `skip` | `0` | Job orders to skip, for paging together with `top`. |
| `order` | `"accepted_date desc"` | `"accepted_date desc"` (newest first), `"begin_date asc"`, or `"search.score() desc"`. |
| `proxyConfiguration` | Apify residential, U.S. | Keep the country set to the United States. DOL only serves U.S. traffic. |

## Output

One record per job order, with housing nested inside it: the primary site in `Housing`, and
every extra site as an element of `Additional Housing`. A real H-2A posting, with the
employer's email and phone replaced and long texts shortened:

```json
{
    "Case Number": "H-300-26232-183154",
    "Job Post URL": "https://seasonaljobs.dol.gov/jobs/H-300-26232-183154",
    "Job Title": "Farmworkers and Laborers",
    "Employer": "South Harvesting LLC (FL-1)",
    "Employer Email": "employer@example.com",
    "Employer Phone": "+15555550100",
    "Employment Type": "TEMPORARY",
    "First Date": "10/19/2026",
    "Last Date": "5/31/2027",
    "Job Duties": "This job is SOC Code is 45-2092 Level I. Workers will perform described job...",
    "Worksite City": "Clewiston",
    "Worksite State": "FLORIDA",
    "Pay Rate Min": 15,
    "Pay Unit": "HOUR",
    "Pay Currency": "USD",
    "Number of Workers Requested": "48",
    "Job Classification": "45-2092.00 - Farmworkers and Laborers, Crop, Nursery, and Greenhouse",
    "Number of Hours Per Week": "36",
    "Date Posted": "9/15/2026",
    "Total Workers": "48",
    "H-2A Workers": "48",
    "Employment City": "Clewiston",
    "Employment State": "Florida",
    "Employment County": "Hendry County",
    "Housing": {
        "Address/Location": "1407 Avocado Street",
        "City": "Immokalee",
        "State": "Florida",
        "Postal Code": "34142",
        "County": "Collier County",
        "Total Units": "1",
        "Total Occupancy": "15"
    },
    "Additional Housing": [
        {
            "Address/Location": "1316 Tangerine Street",
            "City": "Immokalee",
            "State": "Florida",
            "Postal Code": "34142",
            "Total Units": "2",
            "Total Occupancy": "28",
            "Information": "Employer leased housing."
        }
    ]
}
```

The full record also carries the worksite address and ZIP code, how to apply, experience
and special requirements, schedule, piece rates and wage notes.

### One row per housing site

The dataset has a second view, `housing`, that turns each `Additional Housing` entry into
its own row, carrying its job's details. Request it with `view=housing`:

```bash
curl "https://api.apify.com/v2/datasets/<DATASET_ID>/items?view=housing&format=csv" \
  -H "Authorization: Bearer $APIFY_TOKEN"
```

---

## Things to design around

**Keys contain spaces and punctuation**, like `"Case Number"` and
`"Work Schedule (Start/End time)"`, so read them with bracket notation, not dot notation.

**Optional fields are left out, not set to `null`.** Contact, experience, schedule,
trade-name and wage-note fields only appear when the job order includes them. Use
`job.get(...)` in Python or optional chaining in JavaScript.

**H-2B job orders have no housing.** Employers aren't required to provide it, so `Housing`
and `Additional Housing` are empty for them, as is `H-2A Workers`.

**New postings arrive daily.** They reach the scraper the day after DOL accepts them, from
about 1 AM U.S. Eastern time. For a daily feed, schedule a run with `lookbackDays: 2` after
that time.

**Deduplicate on `"Case Number"`.** A posting can appear in two consecutive daily runs.

**Each 20 days of history is one more download.** DOL publishes daily files that each cover
20 days, so a large `lookbackDays` costs a little more time per run.

**Status isn't included.** Open `Job Post URL` to check whether a posting is still active.

---

## Pricing

Pay per job order returned, whatever the filters; the current rate is on the Actor's
[Pricing tab](https://apify.com/alkausari_mujahid/dol-seasonal-jobs-scraper/pricing). Set
`top` to cap how many a run can return.

## Use it from an AI assistant

Apify's MCP server lets Claude, Cursor and other MCP clients run this scraper for you. Add
this server URL to your client:

```text
https://mcp.apify.com?tools=alkausari_mujahid/dol-seasonal-jobs-scraper
```

Then ask in plain words, for example "list this month's H-2A job orders in Georgia with
housing for more than 20 workers".

## Schedules and integrations

On Apify you can run a search on a schedule, get a webhook when a run finishes, and send
results to Google Sheets, Airtable, Zapier, Make, Slack or your own endpoint. All of it is
set up from the [Actor page](https://apify.com/alkausari_mujahid/dol-seasonal-jobs-scraper).

---

## Support

Questions, bugs or feature requests: open an issue in the
[Actor's Issues tab](https://apify.com/alkausari_mujahid/dol-seasonal-jobs-scraper/issues), or
email [alkausarimujahid@gmail.com](mailto:alkausarimujahid@gmail.com).

*This is an independent tool. It is not affiliated with or endorsed by the U.S. Department
of Labor. Job order data is published by DOL on [seasonaljobs.dol.gov](https://seasonaljobs.dol.gov).*
