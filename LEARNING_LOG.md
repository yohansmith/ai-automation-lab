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
- What happened:
- Lesson:
