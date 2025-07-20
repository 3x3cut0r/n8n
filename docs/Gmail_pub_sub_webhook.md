# Configure a Google Pub/Sub Topic to trigger a n8n webhook for new Emails in Gmail

1. Create new Google Cloud Project "nodemation" on `https://console.cloud.google.com/`
2. Create Google OAuth2 API Credentials (Client ID and Secret) inside the created Project and configure the following Redirect URLs:
   - https://n8n.3x3cut0r.de/rest/oauth2-credential/callback -> Replace with your own domain for n8n
   - https://developers.google.com/oauthplayground
3. Activate Google Cloud Pub/Sub API (-> should be done automatically in step 4):

```bash
# For manual activation: Open Google Cloud Shell (CLI) in your Google Cloud and enter:
gcloud services enable pubsub.googleapis.com
```

4. Create a pub/sub topic `gmail` with default abo
   - Note your topic name, e.g.: `projects/nodemation-465018/topics/gmail`
5. Edit default Abo:
   - Change to `push` (from `pull`)
   - Set Endpoint-URL: `https://n8n.3x3cut0r.de/webhook/gmail-new-mail`
6. Activate Google Cloud Shell (CLI) and enter:

```bash
# Replace your topic name from above
gcloud pubsub topics add-iam-policy-binding <topic name> \
    --member="serviceAccount:gmail-api-push@system.gserviceaccount.com" \
    --role="roles/pubsub.publisher"
```

7.  Create new "Gmail OAuth2 API" credentials in n8n

    - Enter your Client ID and Secret
    - Enter scope: `https://www.googleapis.com/auth/gmail.modify`
    - Sign in with your Google Account and grant permissions to your inbox

8.  Create a new workflow in n8n (or import my workflow `Gmail - New Email webhook.json` ([Link](https://github.com/3x3cut0r/n8n/blob/main/workflows/Gmail%20-%20New%20Email%20webhook.json)))
9.  Ensure your workflow has a webhook trigger with this settings:

    - Production URL
    - HTTP Method: `POST`
    - Path: `gmail-new-mail` (same as endpoint-url)
    - Authentication: `None`
    - Respond `Immediately`
      -> Activate your webhook

-> This workflow is quite complex. You need to:

<img width="1962" height="466" alt="Bildschirmfoto 2025-07-20 um 10 55 20" src="https://github.com/user-attachments/assets/3689728b-f645-4e90-83ec-b8d6aaa83ff0" />

- Get historyId from body.message.data (code node)
- Get last historyId from a StaticData field (code node)
  (ensure you have one stored on the first run)
- get history from lastHistoryId (request node)
- Get messageIds from history (code node)
- If you have messageIds, than: (false -> Store new historyId in StaticData)
- Loop over messageIds (loop over node) -> Done: Store new historyId in StaticData
- Get message from messageId (Gmail node)
- If message exists, than: (false -> back to Loop)
- Execute a sub-workflow with the message as input
  (so as not to overload the workflow)
- Back to Loop

10. Send watch() call to Gmail to start watching

    - Visit `https://developers.google.com/oauthplayground/`
    - Click on Settings (OAuth 2.0 configuration):
      Leave default values:
      OAuth flow: `Server-side`
      OAuth endpoints: `Google`
      Authorization endpoint: `https://accounts.google.com/o/oauth2/v2/auth`
      Token endpoint: `https://oauth2.googleapis.com/token`
      Authorization token location: `Authorization header w/ Baerer prefix`
      Access type: `Offline`
      Force prompt: `Consent Screen`
    - Set tick on `Use your own OAuth credentials`
    - Enter your OAuth Client ID: `your Client ID from above`
    - Enter your OAuth Client Secret: `your Secret from above`

    - On Step 1: Select `Gmail API v1` -> `https://www.googleapis.com/auth/gmail.modify`
      or enter manually: `https://www.googleapis.com/auth/gmail.modify`
    - Select `Authorize APIs` and authorize with your Google Credentials for your Gmail Account and follow the assistent

    - On Step 2: Select `Exchange authorization code for tokens` to get your `Refresh token` and `Access token (ya29.XXXXXX)`

    - On Step 3: Set:
      HTTP Method: `POST`
      Request URL: `https://gmail.googleapis.com/gmail/v1/users/me/watch`
      Enter request body and replace your topic name from above: `{ "labelIds": ["INBOX"], "topicName": "projects/nodemation-465018/topics/gmail" }`
      Content/type: `application/json`
    - Send the request -> you should get something like:

      ```bash
      HTTP/1.1 200 OK
      Content-length: 62
      X-xss-protection: 0
      X-content-type-options: nosniff
      Transfer-encoding: chunked
      Vary: Origin, X-Origin, Referer
      Server: ESF
      -content-encoding: gzip
      Date: Wed, 16 Jul 2025 18:31:55 GMT
      X-frame-options: SAMEORIGIN
      Alt-svc: h3=":443"; ma=2592000,h3-29=":443"; ma=2592000
      Content-type: application/json; charset=UTF-8
      {
          "expiration": "1753295515336",
          "historyId": "5635174"
      }
      ```

      - Click `List possible operations` to see possible operations for your workflow
