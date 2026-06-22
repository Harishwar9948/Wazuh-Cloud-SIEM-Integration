# GCP Keyless Authentication — Wazuh + GCP Pub/Sub

Integrating GCP logs into Wazuh using Workload Identity — no JSON key files, no static credentials.

## The Problem with JSON Keys

The traditional approach stores a `service-account-key.json` file on the server.

- Never expires unless manually rotated
- Works from any machine, anywhere in the world
- Leaks through git commits, backups, and log outputs

## The Solution

Attach a GCP service account directly to the Wazuh VM. Google's Metadata Server issues short-lived tokens automatically — there is no key file to steal because there is no key file.

## Architecture


GCP Log Sources → Pub/Sub Topic → GCP Metadata Server → Wazuh Manager
(Audit, VPC Flow)  (Log sink)      (VM identity token)   (subscriber.py)


## Setup

**Step 1 — Create a dedicated service account**
- GCP Console → IAM & Admin → Service Accounts
- Create: `wazuh-pubsub-reader@your-project.iam.gserviceaccount.com`
- Do not create or download any JSON key

**Step 2 — Assign minimum IAM roles**
- `roles/pubsub.subscriber` — Wazuh pulls messages
- `roles/pubsub.publisher` — writes log events to topic

**Step 3 — Attach service account to the Wazuh VM**
- Compute Engine → VM Instances → Edit → Service Account
- Select `wazuh-pubsub-reader` → Save

**Step 4 — Patch subscriber.py**

python
# OLD — requires JSON key file
from google.oauth2 import service_account
credentials = service_account.Credentials.from_service_account_file(
    credentials_file, scopes=[SCOPES]
)

# NEW — uses VM identity, no file needed
import google.auth
credentials, project = google.auth.default(scopes=[SCOPES])


**Step 5 — Create a dummy credentials file**


touch /var/ossec/wodles/gcloud/dummy_creds


> Wazuh's ossec.conf requires a credentials_file path. This empty file satisfies the parser — the patched code never reads it.

**Step 6 — Update ossec.conf**


<wodle name="gcloud-pubsub">
  <enabled>yes</enabled>
  <project_id>your-gcp-project-id</project_id>
  <subscription_name>wazuh-sub</subscription_name>
  <credentials_file>/var/ossec/wodles/gcloud/dummy_creds</credentials_file>
  <max_messages>100</max_messages>
  <interval>1m</interval>
</wodle>


## Troubleshooting


# Verify the VM can reach the metadata server
curl http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token \
  -H "Metadata-Flavor: Google"

# Check Wazuh logs
tail -f /var/ossec/logs/ossec.log | grep gcloud


## Important Note

After every Wazuh upgrade, verify `subscriber.py` has not been reverted to the default version. Keep the patched file backed up.
