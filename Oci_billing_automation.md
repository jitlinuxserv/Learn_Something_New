Here is the complete, corrected 2026 setup. Follow it in order — the order is what prevents the `BucketNotFound` and "no email" failures.

---

Architecture

```text
Upload object → Bucket (Emit Object Events ON) → Events Rule
→ OCI Function (resource principal) → creates PAR link → Notifications topic → Email
```

Everything below must live in one region (example: `ap-mumbai-1`). Mixing regions is the single most common cause of `BucketNotFound`.

---

Step 1 — Collect your four identifiers first

Write these down before touching anything else. Most errors come from guessing them.

| Value | Where to find it |
|---|---|
| Namespace | Profile menu → Tenancy → Object Storage Namespace |
| Bucket name | Storage → Buckets (copy it exactly, case-sensitive) |
| Bucket compartment | Shown on the bucket detail page |
| Region | Top-right region selector |

Important: bucket names are case-sensitive and cannot be renamed. `Daily_compartment_bill` and `daily_compartment_bill` are different buckets.

---

Step 2 — Create the bucket

Storage → Object Storage → Buckets → Create Bucket

- Name: `cost-reports` (lowercase, hyphens — avoid underscores and capitals, they cause copy/paste mistakes)
- Storage tier: Standard (Archive tier cannot create PARs)
- Visibility: Private
- Emit Object Events: ENABLED ← mandatory, the whole flow is dead without it

If the bucket already exists, open it → Edit → tick Emit Object Events.

Optional lifecycle cleanup (two separate rules — one rule cannot do both):
1. `delete-after-30-days` — Action Delete, 30 days, empty prefix
2. `abort-multipart-7-days` — Action Abort Multipart Uploads, 7 days

---

Step 3 — Notifications topic and verified email

Developer Services → Notifications → Create Topic → `object-alert-topic`

Create Subscription → Protocol Email → your address.

Open your inbox and click Confirm subscription. Status must read Active. A Pending subscription silently discards every message.

Test it now: topic page → Publish Message → send. If that test email does not arrive, stop and fix this before continuing — nothing downstream can work.

Copy the Topic OCID (`ocid1.onstopic.oc1...`), not the subscription OCID.

---

Step 4 — Functions application

Developer Services → Functions → Create Application

- Name: `par-notify-app`
- VCN + subnet: the function needs outbound internet access to pull its image:
  - Public subnet → the VCN needs an Internet Gateway and a route rule `0.0.0.0/0 → IGW`
  - Private subnet → the VCN needs a Service Gateway (All Services in Region) and a route rule for the service CIDR
- Enable Logs on the application (Logs tab → Enable). Without this you are debugging blind.

---

Step 5 — Dynamic group

Identity & Security → Domains → your domain → Dynamic Groups → Create

Name: `par-fn-dg`

Rule (use the compartment OCID of the Function, not the bucket):

```text
ALL {resource.type = 'fnfunc', resource.compartment.id = 'ocid1.compartment.oc1..aaaa....'}
```

Never write a rule with only `resource.compartment.id` — that pulls every resource in the compartment into the group.

If your tenancy uses identity domains other than Default, the policy later must reference `dynamic-group '<DomainName>'/'par-fn-dg'`.

---

Step 6 — IAM policies (this is where most setups fail)

Identity → Policies → create in the compartment that is the common parent of the bucket, the topic, and the function. Root compartment is safest while you get it working.

```text
Allow dynamic-group par-fn-dg to read buckets in compartment <comp> where target.bucket.name='cost-reports'
Allow dynamic-group par-fn-dg to read objects in compartment <comp> where target.bucket.name='cost-reports'
Allow dynamic-group par-fn-dg to manage buckets in compartment <comp> where all {request.operation='CreatePreauthenticatedRequest', target.bucket.name='cost-reports'}
Allow dynamic-group par-fn-dg to use ons-topics in compartment <comp> where request.operation='PublishMessage'
```

Critical fact that most outdated docs get wrong: `manage objects` does NOT grant permission to create a pre-authenticated request. PAR creation is `PAR_MANAGE`, which lives under the buckets resource type. That missing statement is the real reason so many people see a 404/403 at `create_preauthenticated_request`.

Also note: Object Storage returns 404 BucketNotFound instead of 403 when you lack permission. A 404 does not mean the bucket is missing — it usually means the policy is wrong, the region is wrong, or the dynamic group does not match.

Plus the service policy so Events can invoke Functions:

```text
Allow service objectstorage-<region> to manage object-family in compartment <comp>
Allow service faas to use ons-topics in compartment <comp>
```

If your tenancy rejects the `where all {request.operation=...}` conditions, fall back temporarily to:

```text
Allow dynamic-group par-fn-dg to manage buckets in compartment <comp>
Allow dynamic-group par-fn-dg to read objects in compartment <comp>
Allow dynamic-group par-fn-dg to use ons-topics in compartment <comp>
```

Then tighten once it works. Wait 5–10 minutes for propagation before testing — policy changes are not instant.

---

Step 7 — The function code

Create the project (Cloud Shell or local):

```bash
fn init --runtime python par-notifier
cd par-notifier
```

`requirements.txt`:

```text
fdk
oci>=2.112.0,<3.0.0
```

The filename is exactly `requirements.txt` — `requirement.txt` is silently ignored and the container then fails to initialize.

`func.py`:

```python
import io
import json
import logging
import os
from datetime import datetime, timedelta, timezone

import oci
from fdk import response

LOGGER = logging.getLogger()
LOGGER.setLevel(logging.INFO)


def reply(ctx, status, payload):
    return response.Response(
        ctx,
        status_code=status,
        response_data=json.dumps(payload),
        headers={"Content-Type": "application/json"},
    )


def handler(ctx, data: io.BytesIO = None):
    try:
        event = json.loads((data.getvalue() if data else b"{}").decode("utf-8"))

        event_type = str(event.get("eventType", ""))
        event_id = str(event.get("eventID", "unknown"))
        ev = event.get("data") or {}
        extra = ev.get("additionalDetails") or {}

        # Object Storage puts these under data.additionalDetails
        object_name = str(ev.get("resourceName", "")).strip()
        bucket_name = str(extra.get("bucketName") or ev.get("bucketName") or "").strip()
        namespace = str(extra.get("namespace") or ev.get("namespace") or "").strip()

        expected_bucket = os.environ["REPORT_BUCKET"].strip()
        topic_ocid = os.environ["TOPIC_OCID"].strip()
        region = os.environ["OCI_REGION"].strip()
        par_hours = int(os.environ.get("PAR_EXPIRY_HOURS", "24"))

        if event_type != "com.oraclecloud.objectstorage.createobject":
            return reply(ctx, 200, {"status": "ignored", "reason": "event type"})

        if not (object_name and bucket_name and namespace):
            raise ValueError(f"Incomplete event payload: {event}")

        if bucket_name != expected_bucket:
            return reply(ctx, 200, {"status": "ignored", "reason": "other bucket"})

        signer = oci.auth.signers.get_resource_principals_signer()
        os_client = oci.object_storage.ObjectStorageClient({"region": region}, signer=signer)

        # Clear checkpoint: a 404 here is permissions/region/name, not PAR logic
        os_client.get_bucket(namespace_name=namespace, bucket_name=bucket_name)

        expires = datetime.now(timezone.utc) + timedelta(hours=par_hours)
        safe_id = "".join(c for c in event_id if c.isalnum() or c in "-_")[-40:]

        par = os_client.create_preauthenticated_request(
            namespace_name=namespace,
            bucket_name=bucket_name,
            create_preauthenticated_request_details=oci.object_storage.models.CreatePreauthenticatedRequestDetails(
                name=f"par-{safe_id}",
                access_type="ObjectRead",
                object_name=object_name,
                time_expires=expires,
            ),
        ).data

        # access_uri is a relative path; prepend the service endpoint
        download_url = os_client.base_client.endpoint.rstrip("/") + par.access_uri

        ons = oci.ons.NotificationDataPlaneClient({"region": region}, signer=signer)
        ons.publish_message(
            topic_id=topic_ocid,
            message_details=oci.ons.models.MessageDetails(
                title="New file available in OCI bucket",
                body=(
                    f"File: {object_name}\n"
                    f"Bucket: {bucket_name}\n\n"
                    f"Download link (expires in {par_hours} hours):\n{download_url}\n\n"
                    "Anyone with this link can download the file until it expires."
                ),
            ),
            message_type="RAW_TEXT",
        )

        LOGGER.info("Notified for %s (event %s)", object_name, event_id)
        return reply(ctx, 200, {"status": "sent"})

    except Exception as exc:
        LOGGER.exception("PAR notification failed")
        return reply(ctx, 500, {"status": "error", "message": str(exc)})
```

Three corrections against most published examples:
- `par.data.full_path` does not exist — use `access_uri` joined to the client endpoint
- event fields live in `data.additionalDetails`, not at the top of `data`
- never hardcode `time_expires` to a far-future date; a leaked PAR is a permanent public download link

---

Step 8 — Deploy

```bash
fn use context <region>
fn update context oracle.compartment-id <compartment-ocid>
fn update context registry <region-key>.ocir.io/<namespace>/par-fns
docker login <region-key>.ocir.io      # username: <namespace>/<username>, password: auth token
fn -v deploy --app par-notify-app
```

Use double hyphens: `--app`. PDFs and web pages often convert this to an em-dash, which breaks the command.

---

Step 9 — Function configuration variables

Functions → `par-notify-app` → `par-notifier` → Configuration. Add, with no quotes around values:

```text
REPORT_BUCKET     = cost-reports
TOPIC_OCID        = ocid1.onstopic.oc1..aaaa...
OCI_REGION        = ap-mumbai-1
PAR_EXPIRY_HOURS  = 24
```

---

Step 10 — Events rule

Observability & Management → Events Service → Rules → Create Rule

- Condition type: Event Type
- Service Name: Object Storage
- Event Type: Object – Create
- Add second condition: Attribute → `bucketName` → `cost-reports`
- Action: Functions → your compartment → `par-notify-app` → `par-notifier`

The rule must be created in the same region as the bucket.

---

Step 11 — Test in this exact order

1. Publish a test message from the topic → email arrives → ONS is proven good.
2. Upload any file to the bucket.
3. Within ~60 seconds you should receive the PAR email.
4. Open the link in a private brow
