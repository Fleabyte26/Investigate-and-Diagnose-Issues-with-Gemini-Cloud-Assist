# OCBL461 - Investigate and Diagnose Issues with Gemini Cloud Assist

Run everything in Cloud Shell. Paste one block at a time and read the line under it before moving on. The Gemini Cloud Assist parts are clicks in the console; they are listed as numbered steps.

## Known lab traps/bugs (read first)

1. The region is different in every lab instance. It is shown in the lab's left panel as "GCP Region". The Setup block asks you for it.
2. STOP before Task 4: click the Task 3 check first. The Task 4 redeploy turns authentication back on, which removes the public access that the Task 3 check looks for.
3. The lab's index.js is written with an unquoted heredoc, so the port number is silently dropped from the startup message. The version here prints it correctly. It behaves the same otherwise.
4. The lab keeps values in shell variables. This guide saves them to files in your home folder, and writes the real URL and project into load-test.sh, so a new terminal never breaks anything.
5. The load test holds the terminal for several minutes. The first run is supposed to fail with "Maximum errors reached". In rare cases it passes; run it again.
6. The alert policy name in Task 6 must be exactly: hello-app service scaled
7. The checker lags. If a check fails right after a step, wait 30-60 seconds and click again.

## Setup

Paste this. It stops and asks one question: type the GCP Region from the lab's left panel (for example us-west1) and press Enter.

```bash
cat > ~/askregion.py <<'EOF'
import os, re
while True:
    v = input("GCP Region from the lab panel: ").strip().split("=")[-1].strip().strip("'").strip('"')
    if re.fullmatch(r"[a-z]+-[a-z]+[0-9]+", v):
        break
    print("That does not look right, try again.")
open(os.path.expanduser("~/region"), "w").write(v + "\n")
print("region saved:", v)
EOF
python3 ~/askregion.py
```

You should see: region saved: and your region.

```bash
gcloud config get-value project 2>/dev/null > ~/project
gcloud config get-value account 2>/dev/null > ~/student
cat ~/project ~/student ~/region
```

You should see three lines: your project ID, your student email, your region.

## Task 1. Configure your environment and account

```bash
gcloud services enable cloudaicompanion.googleapis.com cloudasset.googleapis.com cloudresourcemanager.googleapis.com geminicloudassist.googleapis.com logging.googleapis.com monitoring.googleapis.com
gcloud projects add-iam-policy-binding "`cat ~/project`" --member "user:`cat ~/student`" --role=roles/cloudaicompanion.user --condition=None --quiet > /dev/null
gcloud projects add-iam-policy-binding "`cat ~/project`" --member "user:`cat ~/student`" --role=roles/geminicloudassist.investigationCreator --condition=None --quiet > /dev/null
gcloud projects get-iam-policy "`cat ~/project`" --flatten="bindings[].members" --filter="bindings.members:`cat ~/student`" --format="value(bindings.role)" | grep -E "cloudaicompanion|geminicloudassist"
```

You should see two roles: roles/cloudaicompanion.user and roles/geminicloudassist.investigationCreator.

Check my progress: Enable relevant APIs, and set IAM roles

## Task 2. Develop and deploy a simple Node.js application

Create the app. It answers "Hello Learner!" on the home route.

```bash
mkdir -p ~/hello-app; cd ~/hello-app
cat > index.js <<'EOF'
const express = require("express");
const app = express();
const port = process.env.PORT || 8080;

app.get('/', (req, res) => {
    console.log('Received a request.');
    res.send("Hello Learner!");
});

//Start the server
app.listen(port, () => console.log("hello-app listening on port " + port + "!"));
EOF
cat > package.json <<'EOF'
{
    "name": "hello-app",
    "version": "1.0.0",
    "description": "",
    "main": "index.js",
    "keywords": [],
    "author": "",
    "license": "Apache-2.0",
    "dependencies": {
        "express": "^4.17.2"
    }
}
EOF
node --check index.js && echo syntax ok; ls
```

You should see: syntax ok, then index.js and package.json.

Enable the build APIs and deploy. The service is deployed private on purpose (no unauthenticated access). This takes a few minutes.

```bash
cd ~/hello-app
gcloud services enable run.googleapis.com artifactregistry.googleapis.com cloudbuild.googleapis.com
gcloud run deploy hello-app --region="`cat ~/region`" --no-allow-unauthenticated --source . --quiet
gcloud run services describe hello-app --region="`cat ~/region`" --format="value(status.url)" > ~/url
cat ~/url
```

You should see a Service URL ending in run.app. If the deploy fails on a permission error, wait a minute and paste the block again.

Check my progress: Develop and deploy an application to Cloud Run.

## Task 3. Run the app and investigate issues

Call the app with no credentials.

```bash
curl -s "`cat ~/url`" | head -5
```

You should see an error page: 403 Forbidden. That is the problem you now investigate.

Investigate it with Gemini Cloud Assist (console):

1. Navigation menu > Cloud Run > Services > hello-app > Logs tab.
2. Expand the log entry with the warning icon and read its textPayload.
3. At the top of that entry click Investigate log.
4. Click Enable Gemini Cloud Assist at no cost, then Start chatting.
5. Read the explanation: the request had no valid credentials, and the service requires authentication.

Fix 1: send credentials with the request.

```bash
curl -s -H "Authorization: Bearer `gcloud auth print-identity-token`" "`cat ~/url`"; echo
```

You should see: Hello Learner!

Fix 2: allow public access (console, as the lab does it):

1. On the hello-app service page click Security.
2. Under Authentication select Allow public access.
3. Click View diff and redeploy, then Deploy changes. Wait until it finishes.

```bash
curl -s "`cat ~/url`"; echo
```

You should see: Hello Learner! with no credentials. If you still see 403, wait 30 seconds and paste again.

Check my progress: Cloud Run service authentication issue is investigated and resolved.

STOP: do not start Task 4 until this check is green. Task 4 makes the service private again.

## Task 4. Modify the application, and create a load test

Rewrite index.js with a new /load route that burns CPU, then redeploy.

```bash
cd ~/hello-app
cat > index.js <<'EOF'
const express = require("express");
const app = express();
const port = process.env.PORT || 8080;

app.get('/', (req, res) => {
    console.log('Received a request.');
    res.send("Hello Learner!");
});

// Use a new route to simulate a heavy computational load
app.get('/load', (req, res) => {
    console.log('Received a request on the load route.');
    // Add new constants to help define load
    const loopSize = 500000000;
    const logInterval = 100000000;

    // Use a for loop to create a significant compute load
    let result = 0;
    for (let i = 0; i < loopSize; i++) {
        result = Math.sqrt(i);

        // Log calculations at the set interval
        if (i % logInterval == 0) {
            console.log('The square root of ' + i + ' is ' + result);
        }
    }

    res.send("Computation complete.");
});

//Start the server
app.listen(port, () => console.log("hello-app listening on port " + port + "!"));
EOF
node --check index.js && echo syntax ok
```

You should see: syntax ok

```bash
cd ~/hello-app
gcloud run deploy hello-app --region="`cat ~/region`" --no-allow-unauthenticated --source . --quiet
curl -s -H "Authorization: Bearer `gcloud auth print-identity-token`" "`cat ~/url`/load"; echo
```

You should see: Computation complete. (the curl takes a few seconds).

Now create the load test. It is the lab's script, pasted in three parts. DLR is a stand-in that the last block turns into the shell's variable sign.

```bash
set +H
cat > ~/load-test.sh <<'EOF'
#!/bin/bash

#Declare relevant variables
URL="APP_URL_X/load"
CONCURRENCY=80
LOOPS=5
ERROR_COUNT=0
MAX_ERRORS=3
TOKEN=DLR(gcloud auth print-identity-token)
LOG_NAME="projects/PROJECT_X/logs/run.googleapis.com%2Frequests"

# Define a function to execute when SIGINT is received
cleanup_and_exit() {
    echo -e "\nCtrl+C detected. Performing cleanup and exiting..."
    for pid in "DLR{background_pids[@]}"; do
        if kill -0 "DLRpid" 2>/dev/null; then # Check if process still exists
            echo "Killing background process with PID: DLRpid"
            kill "DLRpid" # Send SIGTERM
        fi
    done
    exit 1 # Exit with a non-zero status to indicate abnormal termination
}

# Trap the SIGINT signal and call the cleanup_and_exit function
trap cleanup_and_exit SIGINT

echo "Sending DLRCONCURRENCY concurrent requests, DLRLOOPS times to DLRURL..."

EOF
wc -l ~/load-test.sh
```

You should see: 28

```bash
cat >> ~/load-test.sh <<'EOF'
# Use nested loops to run the load test
for i in DLR(seq 1 DLRLOOPS); do
    echo "--- Starting batch DLRi of DLRLOOPS ---"
    # The first curl command sends a concurrent load of requests
    for j in DLR(seq 1 DLRCONCURRENCY); do
        curl -s -o /dev/null -H "Authorization: Bearer DLRTOKEN" "DLRURL" &
    done
    echo "Requests sent"

    # Print a dot every half second, as a background job
    while true; do
        echo -n "."
        sleep 0.5
    done &

    # Get the process ID of that background job
    pid=DLR!
    background_pids+=(DLRpid)

    # Send a second curl command as a "canary" that checks for errors
    STATUS_CODE=DLR(curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer DLRTOKEN" "DLRURL")

    # Kill the background process when the second curl command completes
    kill "DLRpid"
    echo ""

EOF
wc -l ~/load-test.sh
```

You should see: 54

```bash
cat >> ~/load-test.sh <<'EOF'
    # Check the canary's status code and print a message
    if [[ "DLRSTATUS_CODE" -eq "200" ]]; then
        echo "Sent test request. No error detected! Status Code: 200"
    else
        echo "Sent test request. Error detected. Status Code: DLRSTATUS_CODE"
        ERROR_COUNT=DLR((ERROR_COUNT + 1))
    fi

    # Stop the script if the maximum number of errors is reached
    if [[ "DLRERROR_COUNT" -ge "DLRMAX_ERRORS" ]]; then
        echo "Maximum errors reached. Halting load test."
        exit 1
    fi
done

echo "Waiting for remaining requests..."
wait

echo "Success! Load test complete."
gcloud logging write DLRLOG_NAME "Success! Load test complete."
EOF
sed -i 's/DLR/\x24/g' ~/load-test.sh
sed -i "s|APP_URL_X|`cat ~/url`|; s|PROJECT_X|`cat ~/project`|" ~/load-test.sh
chmod +x ~/load-test.sh
bash -n ~/load-test.sh && echo syntax ok
grep -c -E "DLR|APP_URL_X|PROJECT_X" ~/load-test.sh
grep -n -E "^URL=|^LOG_NAME=" ~/load-test.sh
```

You should see: syntax ok, then 0, then the URL line with your run.app address and the LOG_NAME line with your project ID.

Check my progress: Cloud Run app is modified and re-deployed.

## Task 5. Run the load test, and investigate issues

Run the load test. It sends 80 requests at once, five times. Let it finish (several minutes).

```bash
~/load-test.sh
```

You should see errors (status 429 or 500) and then: Maximum errors reached. Halting load test. That failure is expected.

Investigate it with Gemini Cloud Assist (console):

1. Cloud Run > hello-app > Logs tab.
2. In the Severity menu select Warning. You should see GET 429 entries (maybe GET 500 too).
3. Expand one entry and click Investigate log.
4. Read the findings: the service hit its maximum instance limit (3 by default in this lab).

Raise the limit.

```bash
gcloud run services update hello-app --region="`cat ~/region`" --max-instances=10 --quiet
gcloud run services describe hello-app --region="`cat ~/region`" --format=yaml | grep -i -E "maxScale|max-instance"
```

You should see a line with 10. Also check the service page in the console: Scaling should show Max: 10. If it still shows 3 after a few minutes, click the pencil next to Scaling, set Maximum number of instances to 10, and Save.

Run the load test again.

```bash
~/load-test.sh
```

You should see five batches with "No error detected! Status Code: 200" and then: Success! Load test complete. If it still halts, paste the update block again with 20 in place of 10 and rerun.

Check my progress: Load test completed successfully.

## Task 6. Investigate issues with Cloud Monitoring alerting

Create the alert policy (console):

1. Navigation menu > Monitoring > Alerting > Create policy.
2. Click Select a metric. Under Active Resources click Cloud Run Revision, then Container, then Instance Count. Click Apply.
3. Leave the defaults and click Next.
4. Condition type: Threshold. Threshold value: 1. Click Next.
5. Turn off Use notification channel.
6. Severity: Warning.
7. Alert policy name: hello-app service scaled. Click Next.
8. Click Create policy.

Trigger it by scaling the service.

```bash
~/load-test.sh
```

Investigate the alert (console):

1. Monitoring > Alerting. There should be at least one incident from your policy (it can take a minute or two to appear).
2. Click the incident summary link.
3. Scroll to the Logs section, expand a warning or error entry, and click Investigate log.
4. Read Gemini's analysis of why the alert fired.

Check my progress: Configure the alert policy.

## Quiz

This lab has no quiz.
