# Lab 4 · Multi-service app with Compose

GitHub user: chamalka001 · Reference files used (Lab 2 fallback): no

<!-- Keep every heading as it is: the grader finds the sections by name. Fill in every blank (four underscores) and your GitHub user name: check.sh writeup flags anything left. -->

## 1 · Architecture

```mermaid
flowchart LR
  B["Browser on your laptop"]
  subgraph front["network: front"]
    W["web: nginx :80"]
  end
  A["api :5000 · networks: front, back"]
  subgraph back["network: back"]
    D[("db :5432")]
  end
  V[("volume pgdata")]
  B -->|"HTTP localhost:8080"| W
  W -->|"HTTP api:5000"| A
  A -->|"PostgreSQL db:5432"| D
  D --- V
```

Which arrow crosses from your laptop into Docker: Browser -> web, through localhost:8080

## 2 · Predictions and what happened

Part 1: fill in "I predicted" (the letter and a one-line reason) before you write any YAML. Part 3: fill in the rest. A wrong prediction that you explain earns full marks.

| # | I predicted | What I saw (command → the deciding output line) | Match? | If not: why |
|---|-------------|--------------------------------------------------|--------|-------------|
| P1 | b) nothing - web is only on the front network, and db is only on back, so web can't find the name db| docker compose exec web getent hosts db → printed nothing (but getent hosts api printed 172.19.0.2) | yes, it matches | |
| P2 | b) refused - db has no published port, so nothing on laptop listens on 5432 | nc -zv -w 3 localhost 5432 → Connection refused | yes, matched | |
| P3 | a) still there - the data is stored in a named volume, and down doesn't delete volumes | curl POST "survive me", then docker compose down and up -d --wait, then curl /api/todos → still lists "chamalka001: survive me" (id 3) | yes, matched | |
| P4 | c) the table is gone too - -v deletes the volume, so the whole database is empty again | docker compose down -v, up -d --wait db, select count(*) from todos → ERROR: relation "todos" does not exist | yes, it matches | |
| P5 | a) healthy · 500 - api’s own health check doesn’t need the database, but the request does | docker compose stop db, then docker compose ps → api still "Up (healthy)"; curl -i localhost:8080/api/todos → HTTP/1.1 500 Internal Server Error | yes, it matches | |
| P6 | b) fail: the name isn’t found - browser is on laptop, outside Docker’s network, so it can’t resolve api | reasoning from P1: my browser is on my laptop, not on a Docker network, so it can’t resolve api | yes, matched | |

## 3 · Backup and restore

```text
$ mkdir -p backup
$ docker compose exec -T db pg_dump -U todo -d todo > backup/todo.sql
$ grep 'survive me' backup/todo.sql
3       chamalka001: survive me f       2026-10-20 09:46:54.872992+00
$ docker compose down -v
[+] down 6/6
 ✔ Container course-app-web-1 Removed
 ✔ Container course-app-api-1 Removed
 ✔ Container course-app-db-1  Removed
 ✔ Volume course-app_pgdata   Removed
 ✔ Network course-app_back    Removed
 ✔ Network course-app_front   Removed
$ docker compose up -d --wait db
 ✔ Container course-app-db-1 Healthy
$ docker compose exec db psql -U todo -d todo -c 'select count(*) from todos'
ERROR:  relation "todos" does not exist
$ docker compose exec -T db psql -U todo -d todo < backup/todo.sql
SET
...
CREATE TABLE
ALTER TABLE
CREATE SEQUENCE
COPY 3
ALTER TABLE
$ docker compose up -d --wait
 ✔ Container course-app-db-1  Healthy
 ✔ Container course-app-api-1 Healthy
 ✔ Container course-app-web-1 Healthy
$ curl -s localhost:8080/api/todos
[{"id":1,"title":"Do the assignments",...},{"id":2,"title":"chamalka001: first",...},{"id":3,"title":"chamalka001: survive me",...}]
```

Why the restore must run before `api` starts: when api starts on an empty database it creates the todos table itself, so the restore would then fail with "relation todos already exists". The restore needs the database running (so psql can connect), but api must not be running yet.

## 4 · Compose clinic

### c1 (worked example)
- Error line (copied): `connect ECONNREFUSED 127.0.0.1:5432`
- Checkpoint: port, inside `api`
- Fix: `DB_HOST: db`
- Why this fix, and not the tempting one: inside `api`, `localhost` is `api` itself, and nothing listens on 5432 there. Docker's DNS answers `db` with the database container's address. Publishing `db`'s port would "work" by opening a door into the database.
- Hints used: 0

<!-- Copy the block below once for every mission you solved and change cN to c2, c3 … Then delete the cN block itself. -->

### c2
- Committed fix: `lab4/clinic/c2/`
- Error line (copied): `0.0.0.0:3000->8102/tcp` in `docker compose ps` (and nothing answered on localhost:8102)
- Checkpoint (name / path / port / app / config / data): port
- Fix: `ports: "8102:3000"`
- Why this fix, and not the tempting one: the ports format is HOST:CONTAINER. nginx listens on 3000 inside the container, so the container side must be 3000 and the laptop side 8102. The numbers were swapped. Editing nginx.conf to listen on 8102 is not allowed, and it would hide the real mistake.
- Hints used: 0

### c3
- Committed fix: `lab4/clinic/c3/`
- Error line (copied): `host not found in upstream "api" in /etc/nginx/conf.d/default.conf:6`
- Checkpoint (name / path / port / app / config / data): name
- Fix: `networks: [front, back]` on api
- Why this fix, and not the tempting one: Docker's DNS only answers names between containers that share a network. web was on front and api on back, so they shared none. Putting web on back would make the name work, but then web could reach db directly, which breaks the design. Putting api on both networks makes it the only bridge.
- Hints used: 0

### c4
- Committed fix: `lab4/clinic/c4/`
- Error line (copied): `Failed to fetch` on the page, and `net::ERR_...` for `todos` in the browser's Network tab, from app.js:8 (curl to /api/todos answered 200)
- Checkpoint (name / path / port / app / config / data): app
- Fix: in `public/app.js`: `const API = '/api/todos';`
- Why this fix, and not the tempting one: the JavaScript runs in the browser on my laptop, which is not on any Docker network, so it can't resolve the name api. A relative path goes to nginx, and nginx forwards /api/ to api inside Docker. Publishing api's port 5000 would work, but it opens a second public door, and only web should have one.
- Hints used: 0

### c5
- Committed fix: `lab4/clinic/c5/`
- Error line (copied): `curl -s localhost:8105/api/todos` printed `[]` after `docker compose down` and `up`, and `docker volume ls` showed a second random-name volume
- Checkpoint (name / path / port / app / config / data): data
- Fix: added `- pgdata:/var/lib/postgresql/data` under db and a `volumes: pgdata:` block at the bottom
- Why this fix, and not the tempting one: without a named volume, Postgres writes to an anonymous volume that gets a new random name when the container is recreated, so the old data is left behind. A named volume keeps its name across down and up. A ./data bind mount would also keep the data, but it ties the stack to one folder on one machine and can cause permission problems.
- Hints used: 0

## 5 · Run in one command

```bash
git clone https://github.com/chamalka001/turiba-devops-labs.git && cd turiba-devops-labs
cp .env.example .env
docker compose up -d --wait
```

## 6 · Answers

### A1
In c1 the API logged `ECONNREFUSED 127.0.0.1:5432`. Which container was `127.0.0.1`, and why does `db` work instead?
127.0.0.1 was the api container itself, because inside a container localhost means that container. Nothing listens on port 5432 inside api, so the connection was refused. The name db works because api and db share a network, and Docker's DNS turns the service name db into the IP address of the database container, where PostgreSQL listens on 5432.

### A2
Trace `POST /api/todos` from your browser to PostgreSQL: each hop, with the name and port used. Why does `db` have no `ports:`?
Browser → localhost:8080 (the published port, forwarded to port 80 of the web container) → nginx sees the path /api/ and uses proxy_pass http://api:5000 (web finds api by name on the front network) → api container on port 5000 → api connects to db:5432 over the back network → PostgreSQL stores the todo, and the answer goes back the same way. db has no ports: because only api needs to reach it, and containers talk to each other over Docker networks without published ports. Publishing 5432 would open the database to my laptop and my network. My test P2 showed that the laptop gets connection refused.

### A3
`docker compose down` kept your todos, and `docker compose down -v` deleted them. Where were they stored, and what exactly does `-v` remove?
The todos were stored in the named volume course-app_pgdata, which is mounted in db at /var/lib/postgresql/data. docker compose down removes the containers and networks, but not named volumes, so the new db container mounted the same volume and the data was still there. docker compose down -v also removes the named volumes declared in the compose file, here pgdata, so the database started empty and even the todos table was gone.

### A4
With `db` stopped, `docker compose ps` showed `api` as healthy, but `/api/todos` answered 500. Explain both. What does `service_healthy` protect against, and what not?
The health check of api only asks whether the API process itself answers, it does not query the database. So api stays healthy while db is stopped, but /api/todos needs a database query, which fails, so it answers 500. Healthy means "ready by its own check", not "working end to end". service_healthy protects the start: api only starts after the db health check passes, so it never starts against a database that is not ready. It does not protect later: if db stops after api is running, Compose does not stop or restart api, and requests simply fail.
