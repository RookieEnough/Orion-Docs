# The Serverless Community Engine
## The "Impossible" Features

> [!IMPORTANT]
> **Paradox:** How do you store dynamic user data (XP, Ranks, Avatars) in a system designed to be static and read-only?
> **Answer:** You don't use a database. You use a message queue.

---

### A. The Ghost Relay (Serverless Leaderboard)

**The Challenge:**
We wanted a global leaderboard for user contributions. Usually, this requires a Postgres DB + a Node.js server. We had $0 and a static file system.

**The Architecture:**

1.  **Client (The Signature):**
    When a user claims XP, the app generates a cryptographic signature of the payload using a local key.
    ```json
    { "user": "OrionFan", "xp": 500, "sig": "sha256..." }
    ```

2.  **The Relay (The Gatekeeper):**
    A Cloudflare Worker (`submission_relay.js`) receives the POST request.
    *   It verifies the signature.
    *   It rate-limits the IP.
    *   *Crucially:* It does **NOT** write to a database.

3.  **The Storage (The Trick):**
    The Worker uses the GitHub API to open a **GitHub Issue** in a private repo.
    *   **Title:** `[Leaderboard] Submission: OrionFan`
    *   **Body:** JSON payload.

4.  **The Processor (GitHub Actions):**
    An Action (`process_leaderboard.yml`) triggers on `issues: opened`.
    *   It parses the JSON from the issue body.
    *   It calculates the new rankings.
    *   It updates a static `leaderboard.json` file on the *Ghost Branch*.
    *   It closes the issue.

5.  **The Read:**
    The Orion App simply reads `leaderboard.json` from the CDN.

**Why it's clever:**
We are using GitHub Issues as an **Asynchronous Message Queue** and the repo itself as a **NoSQL Database**.
*   Cost: $0.
*   Scalability: Handled by GitHub.

---

### B. "Issue-Ops" App Submission Pipeline

**The Challenge:**
We needed a way for non-technical users to submit new apps to the store.
*   *Traditional Way:* Build a web portal, backend, auth system, database. (Too expensive/complex).
*   *Git Way:* Ask users to Fork & PR. (Too hard for normal people).

**The Solution:**
We built a UI that drives Git automation.

#### 1. The UI (Client Side)
The User fills out a form in the Orion App ("Add App").
Instead of sending an API request, the app constructs a URL:
`https://github.com/Orion/Store/issues/new?title=New+App&body=...`

The body contains a hidden YAML/JSON block:
```yaml
<!-- config-start
name: "Super App"
package: "com.super.app"
repo: "https://github.com/super/app"
config-end -->
```

#### 2. The Trigger (Human Review)
The user clicks the link and hits "Submit Issue".
Maintainers see the issue. They check the app.
If it's good, they just add the **"approved"** label.

#### 3. The Automation (The Robot)
A GitHub Action (`app_submission.yml`) fires when the "approved" label is added.
1.  **Regex Parse:** It extracts the YAML block from the issue body.
2.  **Validate:** Checks if the package already exists.
3.  **Commit:** It effectively runs `git commit` to add the app to `apps.json` in the `main` branch.
4.  **Close:** It comments "Welcome to Orion!" and closes the issue.

**Result:**
A fully automated moderation pipeline managed entirely through the GitHub mobile app. We manage the store while sitting on the bus, using labels as buttons.

### C. Governance via Labels
We use a strict Label Policy to control the state machine:
| Label | Action | Bot Trigger |
| :--- | :--- | :--- |
| `submission` | New app awaits review | - |
| `approved` | merges app to store | `app_submission.yml` |
| `rejected` | closes issue with reason | `reject_bot.yml` |
| `update-req` | notifies dev to fix json | `notify_bot.yml` |

This turns GitHub Issues into a **Admin Dashboard** without writing a single line of UI code.

