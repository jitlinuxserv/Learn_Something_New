# Oracle Cloud — Daily Billing Email, 100% From the Console (No Server, No Installation)

**Goal:** Every day, automatically receive an email with your Oracle Cloud (Pay As You Go) cost —
**even when the bill is zero**, so a `0.00` day still lands in your inbox.

**Everything in this guide is done by clicking inside the Oracle Cloud Console.**
No VM, no SSH, no CLI install, no scripts, no cron. Your two running VMs are not touched at all.

```text
  Console: Cost Analysis  ->  Saved report  ->  Daily schedule
                                   |
                                   v
                    Notifications topic (ONS)  ->  Email subscription  ->  your inbox
                                   ^
                    Budget alert (0.01 threshold)  ---- optional safety net
```

**Cost of the whole setup: $0.00** — Cost Analysis, Scheduled Reports, Budgets and the first
1,000,000 Notifications email deliveries per month are all free.

---

## Step 0 — Things to have open before you start

| Item | Where in the Console |
|---|---|
| Home region | Profile menu (top-right) → **Tenancy** → *Home Region* |
| Tenancy admin access | You must be in the `Administrators` group (or have the policy in Step 1) |
| Your email address | The inbox that should receive the daily mail |

> **Everything billing-related lives in your HOME region.** Before each step, check the region
> selector at the top-right of the Console and switch it to your home region (e.g. `India West (Mumbai)`).
> If you create the topic in the wrong region, the schedule will not find it.

---

## Step 1 — (Admins can skip) Allow the services to talk

Only needed if you are **not** a tenancy administrator.

1. Hamburger menu → **Identity & Security → Policies**.
2. Compartment: **root** (the one named after your tenancy) → **Create Policy**.
3. Name: `daily-billing-mail-policy`, Description: `Allow cost reports to email`.
4. Toggle **Show manual editor** and paste:

```text
Allow group Administrators to read usage-report in tenancy
Allow group Administrators to manage usage-budgets in tenancy
Allow group Administrators to manage ons-family in tenancy
Allow group Administrators to manage cloudevents-rules in tenancy
Allow service usage_reports to use ons-topics in tenancy
```

> There is a second, file-based way to do the same thing — the daily report is written into your own
> Object Storage bucket and the file's arrival triggers the mail. See
> [`ORACLE_DAILY_BILL_EMAIL_BUCKET_METHOD.md`](./ORACLE_DAILY_BILL_EMAIL_BUCKET_METHOD.md).
> It is also 100% Console-only, no server.

5. **Create**.

---

## Step 2 — Create the Notifications topic (the "mailing list")

1. Hamburger menu → **Developer Services → Application Integration → Notifications**.
2. Region selector = your **home region**. Compartment = **root**.
3. Click **Create Topic**.
   - **Name:** `daily-billing-mail`
   - **Description:** `Daily Oracle Cloud cost report`
   - **Create**.
4. Open the topic → **Create Subscription**.
   - **Protocol:** `Email`
   - **Email:** your address
   - **Create**.
5. Open your inbox. Oracle sends *"OCI Notifications Service Subscription Confirmation"*.
   Click **Confirm subscription**.
6. Back in the Console, refresh: the subscription state must read **Active** (not *Pending*).
7. Want more recipients? Repeat step 4 for each address — each person confirms their own mail.

> **Test it now:** on the topic page click **Publish Message**, type a title and body, **Publish**.
> If that test mail arrives, the delivery path is working and every later step will work too.

---

## Step 3 — Build the cost report in Cost Analysis

1. Hamburger menu → **Billing & Cost Management → Cost Analysis**.
2. Region selector = home region.
3. Set the report up like this:

| Field | Value |
|---|---|
| **Report type / Metric** | `Cost` (not *Usage*) |
| **Date range** | `Last 7 days` (a rolling range — the schedule re-evaluates it each run) |
| **Granularity** | `Daily` |
| **Group by** | `Service` (optionally add a second group-by of `Compartment`) |
| **Filters** | none — leave empty so the whole tenancy is included |

4. Confirm the chart and the table below it show your two VMs' services
   (`Compute`, `Block Storage`, `Virtual Cloud Network`, …). A `0` row is fine and expected on
   Always Free / no-usage days.
5. Click **Save as** (top of the page) → name it `Daily Bill Report` → **Save**.
   It now appears under **Saved Reports** in the left menu.

---

## Step 4 — Schedule the report to email you every day

1. Still in **Billing & Cost Management**, open **Scheduled Reports**
   (in some tenancies it is the **Schedules** tab inside *Cost Analysis*).
2. Click **Create Schedule**.
3. Fill in:

| Field | Value |
|---|---|
| **Name** | `daily-bill-email` |
| **Description** | `Daily cost mail, sends even when zero` |
| **Compartment** | root |
| **Schedule type / Frequency** | `Daily` |
| **Time of day (UTC)** | `03:00` UTC  → 08:30 IST |
| **Report type** | `Cost` |
| **Query** | choose the saved report `Daily Bill Report` (or re-enter Granularity `Daily`, Group by `Service`) |
| **Date range** | `Last 7 days` |
| **Notification / Output** | select the topic **`daily-billing-mail`** created in Step 2 |
Make sure in advanced manual editor add as below 


Allow service metering_overlay to manage objects in tenancy where all {target.bucket.name='YOUR_BUCKET_NAME', any {request.permission='OBJECT_CREATE', request.permission='OBJECT_DELETE', request.permission='OBJECT_READ'}}

Just a above and replace the bucket name with your created bucket name 


4. Click **Create**.
5. On the schedule's detail page use **Run now** (or **Test**) if the button is offered — the mail
   should arrive in a couple of minutes.

**What you receive:** an email per run containing the cost table for the covered days, per service,
with the currency and totals. Because it is scheduled — not threshold-based — **it is sent every day
regardless of the amount, including days where the total is 0.00.**

> **Why "Last 7 days" instead of "Yesterday":** Oracle finalises usage with a few hours' delay. A
> rolling 7-day window means each mail shows yesterday's figure *and* corrects the previous days, so
> you always see the true live picture.

> **If your tenancy does not show "Scheduled Reports":** it is only in the home region and needs the
> Step 1 policy. Switch region, refresh the browser, then use Step 5 as the alternative.

---

## Step 5 — Alternative / backup: a Budget that emails at the tiniest spend

Use this if Scheduled Reports is unavailable, or keep it alongside as a safety net.

1. **Billing & Cost Management → Budgets → Create Budget**.
2. Fill in:
   - **Target:** `Tenancy` (covers both VMs and everything else)
   - **Name:** `payg-daily-watch`
   - **Monthly budget amount:** a small realistic number, e.g. `10`
   - **Create**.
3. Open the budget → **Alert Rules → Create Alert Rule**:
   - **Threshold metric:** `Actual spend`
   - **Threshold type:** `Absolute amount`
   - **Amount:** `0.01`
   - **Message:** `OCI spend has started for this month`
   - **Email recipients:** your address (or select the `daily-billing-mail` topic)
   - **Create**.
4. Add two more rules at `50%` and `100%` of the budget so climbing costs are caught early.

**Limitation to be clear about:** a budget alert fires **when a threshold is crossed**, not daily, and
it never sends a "today was zero" mail. That is exactly why Step 4 is the primary method and this is
only the backup.

---

## Step 6 — Also get the raw daily cost files (optional, still no server)

Oracle drops detailed CSV cost files into a tenancy bucket every day, free of charge:

1. Hamburger menu → **Billing & Cost Management → Cost and Usage Reports**.
2. Follow the on-page link to the reports bucket (namespace `bling`, bucket = your tenancy OCID).
3. Files appear daily under `reports/cost-csv/`. Download any day directly from the Console.

Nothing to install — this is just a place to double-check any figure that arrives by email.

---

## Step 7 — Make sure the mail keeps arriving

| Check | How |
|---|---|
| Subscription still **Active** | Notifications → topic → Subscriptions column |
| Schedule still **Active** | Billing & Cost Management → Scheduled Reports → *State* |
| Mail not going to spam | Mark the first mail *Not spam*; add `no-reply@notification.<your-region>.oci.oraclecloud.com` to contacts |
| Figures look right | Compare with **Cost Analysis** for the same date |

Two days with no mail at all → open the schedule and press **Run now**; if that fails, re-check the
Step 1 policy and that the topic is in the home region.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No confirmation email | Wrong address or spam filter | Notifications → subscription → **Resend confirmation** |
| Subscription stuck **Pending** | Confirmation link never clicked | Click the link in the mail; it expires — resend if old |
| Topic not listed when creating the schedule | Topic created in the wrong region | Recreate the topic in the **home region** |
| `Scheduled Reports` page missing | Not in home region / missing policy | Switch region; apply Step 1 policy |
| `NotAuthorizedOrNotFound` | Missing usage-report permission | Apply Step 1 policy in the **root** compartment |
| Mail shows 0.00 while VMs run | Always Free shapes, or usage lag of a few hours | Confirm in Cost Analysis; zero is genuinely possible |
| Amounts change next day | Oracle revises recent usage | Normal — the 7-day window shows the corrected values |

---

## What this costs you

| Component | Charge |
|---|---|
| Cost Analysis & saved reports | Free |
| Scheduled Reports | Free |
| Budgets & alert rules | Free |
| Notifications topic | Free |
| Email deliveries | First 1,000,000 / month free |
| Servers / VMs used for this | **None** |

**Total added to your bill: $0.00** — and from tomorrow morning your inbox confirms it every single
day, zero or not. 😊
