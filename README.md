# CS528 HW3

## 1. Overview

This project implements two Python microservices that serve files from Google Cloud Storage and restrict requests based on the `X-country` header. Service 1 runs as a Cloud Function, handles HTTP requests, logs errors, and publishes forbidden requests to Pub/Sub. Service 2 runs locally using service account impersonation, prints these requests, and appends them to a separate GCS log file.

## 2. Project Structure

```text
.
├── README.md
├── LICENSE
├── .gitignore
├── guideline.txt
├── http-client
├── service1/
│   ├── main.py
│   ├── requirements.txt
├── service2/
│   ├── subscriber.py
│   ├── requirements.txt
└── evidence/
    └── phase4/
        ├── http-client-100.txt
        ├── http-client-summary.txt
        ├── get-200.http
        ├── post-200.http
        ├── get-404.http
        ├── put-501.http
        └── cloud-error-logs.json
```

## 3. How to Start

Run commands from the repository root. Local prerequisites are Python 3.10+, Google Cloud CLI, and curl. The provided `http-client` executable requires macOS on Apple Silicon.

### Cloud resources and authentication

This project uses:

| Resource | Value |
|---|---|
| Project | `cs-528-508021` |
| Bucket files | `gs://cs528hw2/pages/1.html` through `12000.html` |
| Service account | `hw3-microservice-sa@cs-528-508021.iam.gserviceaccount.com` |
| Pub/Sub topic | `forbidden-requests` |
| Pull subscription | `forbidden-requests-sub` |
| Log file | `gs://cs528hw2/forbidden_requests/log.txt` |

The topic, subscription, and service account must already exist. Enable billing and the Cloud Functions, Cloud Run, Cloud Build, Artifact Registry, Cloud Storage, Pub/Sub, and IAM Credentials APIs before deployment.

```bash
gcloud auth login
gcloud config set project cs-528-508021
```

The local user needs **Service Account Token Creator** on the service account. The service account needs **Pub/Sub Publisher** on the topic, **Pub/Sub Subscriber** on the subscription, and permission to read files and create/replace the GCS log. The current setup uses **Storage Object User** on `cs528hw2`; this grants access across the bucket, including `pages/`.

Service 2 obtains and refreshes short-lived impersonated tokens through gcloud. No service account key file or `gcloud auth application-default login` is used. The setup steps for a new laptop are below.

### Deploy Service 1

```bash
gcloud functions deploy hw3-file-service \
  --project=cs-528-508021 \
  --gen2 \
  --region=us-central1 \
  --runtime=python312 \
  --source=service1 \
  --entry-point=main \
  --trigger-http \
  --allow-unauthenticated \
  --service-account=hw3-microservice-sa@cs-528-508021.iam.gserviceaccount.com \
  --set-env-vars=BUCKET_NAME=cs528hw2,OBJECT_PREFIX=pages,PUBSUB_TOPIC=projects/cs-528-508021/topics/forbidden-requests
```

Set the endpoint in the terminal used for HTTP requests:

```bash
FUNCTION_URL='https://hw3-file-service-5tn6zthkwq-uc.a.run.app'
```

### Start Service 2 on a new laptop

`service2/subscriber.py` is a long-running local subscriber. It receives forbidden-request messages from Pub/Sub, prints the country, filename, and method, and appends the same information to `gs://cs528hw2/forbidden_requests/log.txt`. It acknowledges a message only after the log is saved (or the event is already present), so failed writes can be retried without losing the message.

The following commands use a macOS/Linux terminal. Install Git, Python 3.10+ with venv support, and the [Google Cloud CLI](https://cloud.google.com/sdk/docs/install), and ensure `git`, `python3`, and `gcloud` are available in your PATH. Service 2 does not require Apple Silicon; that restriction applies only to the supplied `http-client` binary.

**1. Get the project.** Skip cloning if you already have a local copy.

```bash
git clone https://github.com/Lenoisalive/CS528-HW3.git
cd CS528-HW3
```

The existing cloud resources and deployed Service 1 can be reused. Moving Service 2 to another laptop does not require redeploying Service 1 or recreating the bucket, topic, or subscription.

**2. Log in with your own Google account on the new laptop.**

```bash
gcloud auth login
gcloud config set project cs-528-508021
gcloud auth list --filter=status:ACTIVE --format='value(account)'
```

Use the account authorized to impersonate the project's service account. This is a normal gcloud login; do not run `gcloud auth application-default login`.

**3. Configure permissions once, if not already granted.** A project administrator runs these commands. Replace `YOUR_GOOGLE_EMAIL` with the personal account used in step 2.

```bash
gcloud services enable iamcredentials.googleapis.com pubsub.googleapis.com storage.googleapis.com \
  --project=cs-528-508021

gcloud iam service-accounts add-iam-policy-binding \
  hw3-microservice-sa@cs-528-508021.iam.gserviceaccount.com \
  --project=cs-528-508021 \
  --member='user:YOUR_GOOGLE_EMAIL' \
  --role=roles/iam.serviceAccountTokenCreator

gcloud pubsub subscriptions add-iam-policy-binding forbidden-requests-sub \
  --project=cs-528-508021 \
  --member='serviceAccount:hw3-microservice-sa@cs-528-508021.iam.gserviceaccount.com' \
  --role=roles/pubsub.subscriber

gcloud storage buckets add-iam-policy-binding gs://cs528hw2 \
  --member='serviceAccount:hw3-microservice-sa@cs-528-508021.iam.gserviceaccount.com' \
  --role=roles/storage.objectUser \
  --condition=None
```

Token Creator lets your personal account obtain short-lived tokens for the service account. Subscriber and Storage Object User let that service account consume messages and read/create/replace the log. The bucket currently uses an unconditional bucket-level grant because conditional IAM requires Uniform bucket-level access; this grant also permits changes to the HW2 objects.

These permissions belong to identities, not laptops. If you use the same Google account on a new laptop and the roles are already configured, skip the grants above. A different Google account needs its own Token Creator grant.

**4. Install Python dependencies in a virtual environment.** Run from the repository root. The environment is stored outside the repository.

```bash
python3 -m venv "$HOME/.venvs/cs528-hw3-service2"
source "$HOME/.venvs/cs528-hw3-service2/bin/activate"
python -m pip install -r service2/requirements.txt
```

**5. Check impersonation, then start the subscriber.**

```bash
python service2/subscriber.py --check-auth
python service2/subscriber.py
```

The authentication check should print `Impersonation token acquired for: hw3-microservice-sa@cs-528-508021.iam.gserviceaccount.com`. It does not display the token, consume messages, or verify GCS write permissions. The running subscriber should then print:

```text
Listening: projects/cs-528-508021/subscriptions/forbidden-requests-sub
Identity: hw3-microservice-sa@cs-528-508021.iam.gserviceaccount.com
Log: gs://cs528hw2/forbidden_requests/log.txt
```

Python explicitly passes the impersonated credentials to both cloud clients and refreshes tokens when needed. No service account key needs to be copied from the old laptop.

**6. Verify the complete flow.** Keep the subscriber running and send the forbidden-country curl requests in Section 5 from another terminal. Confirm `FORBIDDEN REQUEST` and `Saved to GCS.` appear, then inspect the GCS log. Stop the old laptop's subscriber during this demonstration so the two instances do not compete for messages on the same subscription.

Press Ctrl+C to stop. To restart in a new terminal, enter the repository directory, activate the same virtual environment, and run `python service2/subscriber.py` again. If token acquisition fails, check the login and Token Creator grant; if GCS returns `storage.objects.create` denied, check the service account's bucket permission.

### Run the provided HTTP client

```bash
chmod u+x http-client
./http-client \
  -d hw3-file-service-5tn6zthkwq-uc.a.run.app \
  -b none -w none \
  -n 100 -i 12000 \
  -p 443 -s -r 528 -t 30 -v
```

This sends 100 HTTPS GET requests. `-b none -w none` requests `/number.html`; Service 1 adds the GCS `pages/` prefix. `-v` displays responses. To save a new run, append `> /tmp/hw3-http-client-latest.txt 2>&1` to the command.

The existing `evidence/phase4/` records show **100 HTTP 200 responses before country restrictions were enabled**. With restrictions enabled, client requests carrying forbidden country headers receive 400 instead.

## 4. HTTP Status Codes

| Status | Trigger |
|---|---|
| **200** | GET or POST requests an existing file and passes the country check. |
| **404** | GET or POST requests a missing file and passes the country check. |
| **501** | An unsupported method reaches the function and passes the country check. |

```bash
# GET → 200
curl -i "$FUNCTION_URL/1.html"

# POST → 200
curl -i -X POST "$FUNCTION_URL/" \
  -H 'Content-Type: application/json' --data '{"filename":"1.html"}'

# Missing file → 404
curl -i "$FUNCTION_URL/this_file_does_not_exist.txt"

# Unsupported method → 501
curl -i -X PUT "$FUNCTION_URL/1.html"
```

The application handles PUT, DELETE, HEAD, CONNECT, OPTIONS, TRACE, and PATCH as unsupported methods. Cloud Run blocks CONNECT and TRACE before they reach the function; observed cloud responses were 400 and 405 respectively. Use `curl -I` for HEAD. Application-generated 404 and 501 errors produce both plain-text and structured JSON logs in Cloud Logging.

## 5. Run and Verify Forbidden-Country Requests

The forbidden countries are **North Korea, Iran, Cuba, Myanmar, Iraq, Libya, Sudan, Zimbabwe, and Syria**. Country matching ignores case and surrounding whitespace and happens before GCS access.

With Service 2 running in terminal A, run these commands in terminal B:

```bash
FUNCTION_URL='https://hw3-file-service-5tn6zthkwq-uc.a.run.app'

curl -i -H 'X-country: Iran' "$FUNCTION_URL/1.html"
curl -i -H 'X-country: Syria' "$FUNCTION_URL/3.html"

curl -i -X POST "$FUNCTION_URL/" \
  -H 'Content-Type: application/json' \
  -H 'X-country: Cuba' \
  --data '{"filename":"2.html"}'
```

Each request should return **400** with `Permission denied`. Service 1 publishes a JSON notification to Pub/Sub, and terminal A should display `FORBIDDEN REQUEST`, its country/file/method, and `Saved to GCS.`

Verify the appended records:

```bash
gcloud storage cat 'gs://cs528hw2/forbidden_requests/log.txt' | tail -n 40
```

Match the `Event ID` between terminal output and the log. Service 2 preserves earlier records, uses GCS generation checks to prevent concurrent overwrites, skips duplicate event IDs, and acknowledges messages only after persistence succeeds. A 400 response alone does not prove delivery: verify both the subscriber output and the GCS file.
