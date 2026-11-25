## Day 1 (Mon, 23 Nov 2026): Docker, n8n, first webhook

### Unknown terminology and concepts

- **Volume:** storage that lives outside a container, so data survives when the container is removed. Without one, data disappears with the container.
- **WEBHOOK_URL:** tells n8n what public address to show for webhook URLs. If it's wrong, the URLs shown in the editor don't match where requests must be sent.
- **Webhook:** a URL that waits for another program to send it data (a POST request) and triggers a workflow when it arrives.
- **Test vs production URL:** `/webhook-test/...` only works while the editor is listening and only for one request. `/webhook/...` works any time the workflow is published.
- **Browser vs webhook:** a browser sends GET, my webhook only accepts POST, so I need curl or Postman.
- **Workflow vs agent:** my hello workflow has fixed steps and no LLM, so it is not an agent.

### Experiment 1: restart (`docker compose restart`)

- Prediction: the workflow will still be there, but I'll have to publish it again.
- What happened: n8n showed a connection error for about a minute while starting, then loaded. The workflow was still there and the production webhook worked without republishing.
- Lesson: restarts are not instant on my laptop. Data lives in Postgres and the volumes, not in the container.

### Experiment 2: down then up (`docker compose down`, `docker compose up -d`)

- Prediction: the containers will be removed, but my data (account, workflow, executions) will survive because it is stored in volumes, not inside the containers.
- What happened: `docker compose down` removed both containers and `docker compose ps` showed nothing running. `docker volume ls` still listed `n8n_n8n_data` and `n8n_pg_data`. After `docker compose up -d` and about a minute of startup, my account, workflow and execution history were all still there.
- Lesson: containers are disposable, volumes hold the data. `down` keeps volumes, and only `down -v` deletes them.

### Experiment 3: test URL without listening

- Prediction: n8n will refuse the request with a 404 error, because the test webhook is only registered while I click "Execute workflow" in the editor.
- What happened: `curl -i` returned `HTTP/1.1 404 Not Found` with the JSON message: the requested webhook "hello" is not registered. The hint said to click "Execute workflow" on the canvas, and that in test mode the webhook only works for one call after clicking.
- Lesson: the test URL is only for building and debugging, one request per click. The production URL (`/webhook/...`) works whenever the workflow is published. The `-i` flag in curl shows the status line and headers, which helps when debugging.

### Experiment 4: wrong env variable

- Prediction: if I change POSTGRES_PASSWORD in .env after the database already exists, n8n will fail to connect to Postgres, because the database keeps the password it was created with.
- What happened: after changing one character in the password and running `docker compose up -d`, both containers were recreated. Postgres showed "healthy", but n8n logged `password authentication failed for user "n8n"` five times with increasing retry delays, then crashed with "There was an error initializing DB". Docker kept restarting it (`restart: unless-stopped`).
- Lesson: Postgres only reads POSTGRES_PASSWORD when it first creates the database, so changing it later makes the app and database disagree. The Postgres healthcheck passes anyway because it checks that the server is up, not the password. Read the first error line in `docker compose logs`, not the stack trace. Back up .env before editing it.

### Experiment 5: `docker compose down -v`

- Prediction: - Prediction: the `-v` flag will delete the volumes, so my account, workflow and execution history will be gone and n8n will show the first-time setup screen. My .env file and the JSON in Git will not be affected.
- What happened: `docker volume ls` showed `n8n_n8n_data` and `n8n_pg_data` before `down -v` and neither afterwards. After `up -d`, n8n showed the owner account setup screen with no workflow and no execution history. I recreated the account, imported `01-hello-webhook.json` from the repo, published it, and the production webhook worked again. My .env was untouched.
- Lesson: `down -v` permanently deletes volumes, which means the database and n8n data. Anything not exported to Git is lost. That's why I export workflows to the repo and keep a backup of .env. Never run `down -v` on a real deployment.

---

## Day 2 (Tue 24 to Wed 25 Nov 2026): n8n data model and the ticket normalizer

### Concepts in my own words

- **Item:** n8n passes data between nodes as a list of items. Most nodes run once per item, so 3 items in means 3 runs.
- **Where webhook data lives:** what I send is under `$json.body`. Headers and query parameters are separate.
- **Expressions:** JavaScript inside `{{ }}`. In Edit Fields the top box is the field name (plain text) and the box below is the value (expression). I put an expression in the name box once and got a column called `object Object`.
- **Safe access:** `?.` and `?? ''` stop the workflow crashing when a field is missing.
- **Reaching back to earlier nodes:** `$('Flatten').first().json` reads data from a named node. I need this because Split Out and Aggregate drop the other fields.
- **Split Out and Aggregate:** Split Out turns a list into separate items. Aggregate collects them back into one list.
  - Individual Fields gives `sku: ["A-100","B-200"]`.
  - All Item Data gives `data: [ {sku, qty}, ... ]` with the full objects.
- **Strings vs numbers:** webhook values arrive as strings (`"2"`, `"1500.50"`), so convert with `Number()` before doing maths. Without it, `"2" + "1"` becomes `"21"`.
- **JavaScript vs Python:** n8n runs on JavaScript. I only need a small toolbox: `trim()`, `toLowerCase()`, `split()`, `map()`, `filter()`, `reduce()`, `Number()`, `?.`, `??`.
- **Node types:** triggers start workflows (Webhook), actions do things (Edit Fields, HTTP Request), logic nodes decide (IF, Switch, Merge), and transform nodes reshape data (Split Out, Aggregate, Code).
- **Workflow vs agent:** my workflows have fixed steps and no LLM, so they are not agents. AI nodes and agents come in Week 2.

### Mini-lessons (scratch workflow)

- Webhook, Edit Fields with `.trim()`, Split Out, Aggregate and a Code node.
- Item counts through the belt: Webhook 1, Split Out 2, Aggregate 1.
- A Code node must return an array of `{ json: ... }` objects.

### What I built: Workflow 02, Ticket Normalizer

- **Flow:** Webhook, Flatten, Valid email?, Has items?, Valid priority?, Split Out, Line items, Aggregate, Build response, Respond.
- **Error responses:** 400 `invalid_or_missing_email`, 422 `no_items`, 422 `unknown_priority`.
- **Final Code node:** SLA hours (high 4, medium 24, low 72), tags split into a list, order total, country name and region from a lookup table.
- Tests: 10 payloads in `n8n/tests/`, run in a loop against the production URL. Output saved in `docs/ticket-test-run.txt`.

### Test results (10 payloads)

| File | Case               | Result                            |
| ---- | ------------------ | --------------------------------- |
| 01   | Good, messy values | 200, SLA 4, total 3800            |
| 02   | Already clean      | 200                               |
| 03   | Medium priority    | 200, SLA 24                       |
| 04   | Low priority       | 200, SLA 72                       |
| 05   | Missing email      | 400                               |
| 06   | Bad email          | 400                               |
| 07   | Empty items        | 422 no_items                      |
| 08   | Unknown priority   | 422 unknown_priority              |
| 09   | Unknown country ZZ | 200, country name and region null |
| 10   | Malformed JSON     | 422 from n8n itself               |

### Predictions vs reality

- File 10 (malformed JSON): I predicted 400, but n8n answered 422 "Failed to parse request body". The workflow never ran, so bad JSON is rejected before my logic.
- File 07 (empty items): Split Out on an empty list produces zero items, so the workflow would end with no reply. I added a `Has items?` check first.

### Problems I hit and how I fixed them

- **"No Respond to Webhook node found":** a Webhook set to respond using a Respond to Webhook node needs one in the workflow. Added one.
- **Edit Fields showed "No input data":** the Webhook had not received anything yet. Fix: click Execute workflow, send one curl, then pin the Webhook output.
- **Yellow triangle on Flatten:** "configuration changed, output may change". It's a warning, not an error, and it clears after a full run.
- **Test URL returned 404 on the second curl:** the test URL accepts one request per Execute click. I pasted two commands at once.
- **Country API returned "success false, deprecated":** REST Countries v1 to v4 were shut down. v5 needs a free API key, a new host and a new response shape. My On Error: Continue setting stopped the workflow crashing. I deactivated the HTTP node and used a lookup table in the Code node instead.
- **Edit Fields hint "expression used in fields 1, 2, 3":** a false alarm, because my expressions were in the value boxes.

### Lessons

- External APIs change or disappear. Every API call needs error handling (On Error: Continue) and a fallback.
- Test URL: one request per click, for building. Production URL: after Publish, works any time, and loops can hit it. Edits only take effect on the production URL after republishing.
- Validate cheap things first (email, items, priority) so bad input fails fast with a clear status code.
- Pin the Webhook output to test later nodes without sending curl every time.
- Name nodes clearly (Flatten, Valid email?, Has items?), because later nodes refer to them by name.

### Still to do

- Webhook authentication and duplicate-ticket detection (Day 6).
- Possible upgrade: REST Countries v5 with the key stored as an n8n credential.
