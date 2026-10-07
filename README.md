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

### Start Service 2 locally

`subscriber.py` receives forbidden-request messages from Pub/Sub, prints them, and appends them to `gs://cs528hw2/forbidden_requests/log.txt`.

Assume Service 1 is already deployed and your Google account has permission to impersonate the configured service account. On a new laptop, install Python 3.10+, Git, and Google Cloud CLI, then run:

```bash
# Install dependencies
python3 -m venv "$HOME/.venvs/cs528-hw3-service2"
source "$HOME/.venvs/cs528-hw3-service2/bin/activate"
python -m pip install -r service2/requirements.txt

# Check authentication and start Service 2
python service2/subscriber.py --check-auth
python service2/subscriber.py
```

The program automatically impersonates `hw3-microservice-sa@cs-528-508021.iam.gserviceaccount.com`; no key file or `gcloud auth application-default login` is needed.

Keep it running and send a forbidden-country request from another terminal using Section 5. You should see `FORBIDDEN REQUEST` followed by `Saved to GCS.` Press Ctrl+C to stop.

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
