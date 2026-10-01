# Task 2: Build the Microservices

**Time budget: 35 minutes**

## Goal

Build two real services - an `api` that accepts work over HTTP, and a `worker` that processes it in the
background - communicating asynchronously through Redis rather than the api doing the work itself
inline. This demonstrates the synchronous-vs-asynchronous communication pattern and the
database/queue-per-purpose thinking Module 2 covers conceptually, with real, running code.

**Why asynchronous matters here**: if the api processed each request inline, a slow task would make the
api itself slow to respond to everything else. By handing work off to a separate `worker` via a Redis
queue, the api can respond immediately (`202 Accepted`) regardless of how long the actual work takes -
the same reason real systems use this pattern for anything slower than a few milliseconds.

## Steps

### 2.1: Create the project structure

**In VS Code:** File > Open Folder, create `capstone-local-gitops`, and inside it create two
subfolders: `api` and `worker`.

### 2.2: Write the api service

**In VS Code:** create `api/package.json`:

```json
{
  "name": "capstone-api",
  "version": "1.0.0",
  "description": "Task submission API for the capstone project",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^5.2.1",
    "redis": "^6.2.1",
    "prom-client": "^15.1.3"
  }
}
```

**In VS Code:** create `api/server.js`:

```javascript
const express = require('express');
const { createClient } = require('redis');
const crypto = require('crypto');
const client_prom = require('prom-client');

const REDIS_URL = process.env.REDIS_URL || 'redis://localhost:6379';

const register = new client_prom.Registry();
client_prom.collectDefaultMetrics({ register });
const taskCounter = new client_prom.Counter({
  name: 'tasks_submitted_total',
  help: 'Total number of tasks submitted to the queue',
  registers: [register],
});

const app = express();
app.use(express.json());

let redisClient;
let redisConnected = false;

app.get('/health', (req, res) => {
  // Readiness depends on Redis being reachable - a real dependency check,
  // not just "is this process alive"
  if (redisConnected) return res.status(200).send('healthy');
  res.status(503).send('redis not connected');
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

app.post('/tasks', async (req, res) => {
  const taskId = crypto.randomUUID();
  const task = { id: taskId, text: req.body.text || '' };
  await redisClient.lPush('tasks', JSON.stringify(task));
  taskCounter.inc();
  res.status(202).json({ id: taskId, status: 'submitted' });
});

app.get('/tasks/:id', async (req, res) => {
  const resultStr = await redisClient.hGet('results', req.params.id);
  if (!resultStr) return res.status(202).json({ status: 'processing' });
  res.status(200).json(JSON.parse(resultStr));
});

async function main() {
  redisClient = createClient({ url: REDIS_URL });
  redisClient.on('error', (err) => {
    console.error('Redis error:', err.message);
    redisConnected = false;
  });
  await redisClient.connect();
  redisConnected = true;
  console.log('API connected to Redis');

  app.listen(3000, () => console.log('API listening on port 3000'));
}

main();
```

A few things worth understanding, not just copying:

- **`/health` genuinely checks the Redis connection**, not just "is this process alive." This is a
  real dependency health check - a common and important pattern, since a process can be technically
  running while completely unable to do its job.
- **`POST /tasks` returns immediately with `202 Accepted`** (not `200 OK`) - HTTP's own status code for
  "request accepted for processing, not yet complete." It pushes the task onto a Redis list and returns
  before any actual processing happens.
- **`GET /tasks/:id` polls for a result** - if the worker hasn't finished yet, it returns `202` again
  with `"processing"`; once done, `200` with the actual result. A real frontend would poll this endpoint
  (or use a webhook) rather than blocking on the original request.

### 2.3: Write the worker service

**In VS Code:** create `worker/package.json`:

```json
{
  "name": "capstone-worker",
  "version": "1.0.0",
  "description": "Background task processor for the capstone project",
  "main": "worker.js",
  "scripts": {
    "start": "node worker.js"
  },
  "dependencies": {
    "redis": "^6.2.1"
  }
}
```

**In VS Code:** create `worker/worker.js`:

```javascript
const { createClient } = require('redis');

const REDIS_URL = process.env.REDIS_URL || 'redis://localhost:6379';

async function main() {
  const client = createClient({ url: REDIS_URL });
  client.on('error', (err) => console.error('Redis error:', err.message));
  await client.connect();
  console.log('Worker connected to Redis, waiting for tasks...');

  // eslint-disable-next-line no-constant-condition
  while (true) {
    // brPop blocks for up to 5 seconds waiting for a task, rather than
    // constantly polling Redis in a tight loop - this is the standard
    // pattern for a Redis-backed queue consumer.
    const popped = await client.brPop('tasks', 5);
    if (!popped) continue;

    const task = JSON.parse(popped.element);
    console.log(`Processing task ${task.id}: "${task.text}"`);

    // Simulate real work taking a moment - in a real system this is where
    // actual processing (image resizing, sending an email, calling
    // another service) would happen.
    await new Promise((r) => setTimeout(r, 1000));

    const processedText = task.text.toUpperCase();
    await client.hSet('results', task.id, JSON.stringify({ status: 'done', result: processedText }));
    console.log(`Finished task ${task.id}`);
  }
}

main().catch((err) => {
  console.error('Worker crashed:', err);
  process.exit(1);
});
```

`brPop` **blocks** for up to 5 seconds waiting for a task to appear, rather than constantly polling
Redis in a tight loop burning CPU - this is the standard pattern for a Redis-backed queue consumer. The
1-second `setTimeout` simulates real work taking time - in a real system, this is where actual
processing (resizing an image, sending an email, calling another service) would happen.

### 2.4: Write both Dockerfiles

**In VS Code:** create `api/Dockerfile`:

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY package.json .
RUN npm install --omit=dev
COPY server.js .
EXPOSE 3000
CMD ["node", "server.js"]
```

**In VS Code:** create `worker/Dockerfile`:

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY package.json .
RUN npm install --omit=dev
COPY worker.js .
CMD ["node", "worker.js"]
```

Notice `package.json` is copied and `npm install` run **before** copying the actual source file - this
is a deliberate Docker layer-caching optimization from Module 3: dependencies change far less often than
source code, so Docker can reuse the cached `npm install` layer on rebuilds as long as `package.json`
itself hasn't changed, making iteration much faster.

### 2.5: Write a Docker Compose file to test both services together, locally, before Kubernetes

**In VS Code:** create `docker-compose.yml` in the top-level `capstone-local-gitops` folder (same level
as `api/` and `worker/`):

```yaml
version: "3.9"

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  api:
    build: ./api
    ports:
      - "3000:3000"
    environment:
      REDIS_URL: redis://redis:6379
    depends_on:
      - redis

  worker:
    build: ./worker
    environment:
      REDIS_URL: redis://redis:6379
    depends_on:
      - redis
```

Notice `REDIS_URL` uses `redis` (the service name) as the hostname, not `localhost` - Docker Compose
automatically creates an internal network where each service can reach the others by service name. This
is the same DNS-by-name pattern Kubernetes Services provide, which Task 4 relies on.

### 2.6: Run it

```
docker compose up --build
```

Wait for all three services to show as started in the logs.

### 2.7: Test the full flow, in a second terminal

```
curl http://localhost:3000/health

curl -X POST http://localhost:3000/tasks -H "Content-Type: application/json" -d '{"text":"hello from compose"}'
```

Copy the `id` from the response, then immediately check its status:

```
curl http://localhost:3000/tasks/PASTE-THE-ID-HERE
```

You should see `{"status":"processing"}` - the worker hasn't finished yet. Wait 2 seconds and run the
same command again:

```
curl http://localhost:3000/tasks/PASTE-THE-ID-HERE
```

Now you should see `{"status":"done","result":"HELLO FROM COMPOSE"}`.

### 2.8: Check the metrics endpoint

```
curl http://localhost:3000/metrics | grep tasks_submitted_total
```

You'll use this exact endpoint again in Task 5, once Prometheus is scraping it automatically instead of
you curling it by hand.

## Verify Task 2

- `docker compose up --build` starts all three containers without errors
- Submitting a task returns `202` immediately with a real UUID
- Checking the task's status immediately after shows `"processing"`; checking again after ~2 seconds
  shows `"done"` with the correctly uppercased text
- `/metrics` shows `tasks_submitted_total` with a count matching how many tasks you've submitted

## Solution Reference

`solutions/02-build-microservices/` contains the complete, working version of every file above -
`api/`, `worker/`, and `docker-compose.yml`.

## Cleanup Before Moving On

```
docker compose down
```

We'll rebuild these same images directly for Kubernetes in Task 4 - Compose was only for this local
integration test.
