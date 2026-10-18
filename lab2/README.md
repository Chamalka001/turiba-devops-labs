# Lab 2 · Containerize it, then debug it

- **Image:** `ghcr.io/chamalka001/course-api:lab2` (public, linux/amd64 + linux/arm64)
- **Image index digest:** `sha256:cd4ec2a8fc4315beabf402230a50def529122ca653b733f74baf810dd5f4d60b`
- **Base image digest:** `node:24-alpine@sha256:44fce696f38ba8c987c4db89954051000bbd05ffd8d8659ba15ed4a8602e937f`

## Part 1 · Images and layers

### 1.1 Base images

| Image | Size | Distro | Default user |
|-------|-----:|--------|--------------|
| node:24 | 1.66 GB | Debian GNU/Linux 12 (bookworm) | root (uid 0) |
| node:24-slim | 333 MB | Debian GNU/Linux 12 (bookworm) | root (uid 0) |
| node:24-alpine | 244 MB | Alpine Linux v3.24 | root (uid 0) |

`docker run --rm node:24-alpine id` printed `uid=0(root)`, so the default user is root.

### 1.2 Writable layers

I created `a.html` in container `a1`. It existed only in `a1`, not in `b1`, although both came from `nginx:alpine`. `docker diff a1` showed `A /usr/share/nginx/html/a.html`. `docker ps -s` showed 115 kB for `a1` and 81.9 kB for `b1`, on top of the shared 73.5 MB image. (Docker rejects one-letter names, so I used `a1` and `b1`.)

### 1.3 Deleting a file in a later layer

cow:bad = 65.4 MB · cow:good = 12.9 MB

`docker history cow:bad` showed a 52.4 MB layer from `dd` and a 4.1 kB layer from `rm`. The delete only added a marker.

### 1.4 Build cache

| Build | RUN step CACHED? | Build time |
|-------|------------------|-----------:|
| b · app.txt changed | Yes | 1.387 s |
| c · deps.txt changed | No | 17.702 s |
| d · app.txt changed, wrong order | No | 16.811 s |

### 1.5 Digest

`node:24-alpine` index digest: `sha256:44fce696f38ba8c987c4db89954051000bbd05ffd8d8659ba15ed4a8602e937f`

## Part 2 · The course API image

| Step | Image | Size |
|------|-------|-----:|
| Naive | course-api:naive | 1.75 GB |
| + .dockerignore, npm ci --omit=dev, exec form | course-api:step1 | 1.66 GB |
| Multi-stage, node:24-alpine, non-root | course-api:lab2 | 250 MB |
| Reduction against naive | | 85.7 % |

The "Database not reachable" log line is expected, because the database arrives in a later session.

`docker stop` before the SIGTERM handler: **3.414 s, exit code 137** · after: **0.444 s, exit code 0**. On my Docker Desktop the forced stop took about 3 s, not the 10 s in the guide, but exit code 137 still shows Docker had to kill the process.

The log after the fix ended with: `SIGTERM received, closing the server`

Output of `docker run --rm course-api:lab2 id`:

```text
uid=1000(node) gid=1000(node) groups=1000(node),1000(node)
```

Output of `docker ps` showing (healthy):

```text
CONTAINER ID   IMAGE             COMMAND                  CREATED          STATUS                    PORTS                                         NAMES
701afba822e4   course-api:lab2   "docker-entrypoint.s…"   11 seconds ago   Up 10 seconds (healthy)   0.0.0.0:8080->5000/tcp, [::]:8080->5000/tcp   api
```

`/healthz` returned `{"status":"ok"}`. `ls -la /app` in the final image showed no `.env`, `.git` or `Dockerfile`. After a code change, the `npm ci --omit=dev` step said CACHED. `docker buildx imagetools inspect` listed linux/amd64 and linux/arm64.

## Part 3 · Linux drills

### 3.1 OS files vs kernel

```text
PRETTY_NAME="Ubuntu 24.04.5 LTS"
6.18.54.1-microsoft-standard-WSL2
```

`/etc/os-release` belongs to the Ubuntu container. `uname -r` shows the kernel of my laptop's WSL 2 VM, which all containers share.

### 3.2 sed

```text
chef tools
ansible tools
docker tools
```

### 3.3 Counting 404s

```text
20
```

### 3.4 Permissions

```text
-rwxr-x--- 1 root root 7 Oct 18 10:01 /lab/f
cat: /lab/f: Permission denied
```

Mode 750 gives the owner full access, the group read and execute, and everyone else nothing. `student` is neither the owner nor in the group, so it is denied. Root bypasses permission checks.

### 3.5 Exported variables

```text
child sees:
child sees: staging
```

A plain shell variable stays in the current shell. `export` passes it to child processes.

### 3.6 PID 1

```text
bash
```

PID 1 in `lab` is `bash`. `docker stop lab` took 3.441 s, because bash does not handle SIGTERM as PID 1, so Docker waited and then killed it.

### 3.7 Two ways to reach nginx

```text
172.18.0.2      web
HTTP/1.1 200 OK
```

Inside `web`, `netstat -ltn` showed nginx listening on `0.0.0.0:80` and `:::80`. From the host, `curl -sI http://localhost:8081/` returned `HTTP/1.1 200 OK`.

Inside the Docker network, Docker's DNS resolves `web` to the container's own IP (172.18.0.2), so I connect straight to port 80. From my laptop, the `-p 8081:80` mapping forwards port 8081 to port 80. Both reach the same nginx.

## Part 4 · Broken containers

### lab2-broken:1 (worked example)
- Symptom: exits immediately, `Exited (127)`, log `sh: nodemon: not found`
- Cause: `CMD ["npm","run","dev"]` runs nodemon, a devDependency that `npm ci --omit=dev` leaves out
- Fix: `CMD ["node", "server.js"]`

### lab2-broken:2
- Symptom: `Exited (255)`, log `exec /entrypoint.sh: no such file or directory`, although the file exists and is executable
- Cause: `cat -A` showed `#!/bin/sh^M$`. The file has Windows CRLF line endings, so the interpreter is looked up as `sh^M`
- Fix: convert the script to LF line endings (for example `RUN sed -i 's/\r$//' /entrypoint.sh`) and add `* text=auto eol=lf` to `.gitattributes`

### lab2-broken:3
- Symptom: container is Up and logs `API listening on port 5000`, but `curl localhost:8082` returns an empty reply
- Cause: `netstat -ltn` showed the app listening only on `127.0.0.1:5000`, which port mapping cannot reach
- Fix: listen on `0.0.0.0`, for example `app.listen(PORT, '0.0.0.0')`

### lab2-broken:4
- Symptom: container is Up, but the log says `ECONNREFUSED ::1:5432` and `127.0.0.1:5432`
- Cause: a `.env` with `DB_HOST=localhost` was copied into the image (no `.dockerignore`), so the app looks for the database inside its own container. `docker exec env` showed nothing because the value came from the file
- Fix: add a `.dockerignore` that excludes `.env` and set `DB_HOST` at run time, pointing at the database container by name

### lab2-broken:5
- Symptom: `Exited (1)`, log `EACCES: permission denied, open '/app/data/todos.log'`
- Cause: the app runs as `uid=1000(node)`, but `/app/data` is owned by uid 0 with mode 755
- Fix: `RUN mkdir -p /app/data && chown node:node /app/data` before `USER node`, or write the log to `/tmp`

### lab2-broken:6
- Symptom: `docker stop` took 3.404 s and the exit code was 137
- Cause: `docker exec b6 ps` showed `node server.js` as PID 1 with no SIGTERM handler, so the signal was ignored and Docker sent SIGKILL
- Fix: add a SIGTERM handler that closes the server and exits 0 (or run with `docker run --init`)

## Answers

1. **`cow:bad` contains no `/big.file`, yet it is 50 MB bigger than `cow:good`. Why?** Image layers are read-only. The `dd` layer stored the 50 MB file. The later `rm` layer only added a "deleted" marker, so the bytes stay in the earlier layer. In `cow:good`, the file is created and deleted in one layer, so it is never stored.

2. **Why does the order of `COPY` and `RUN` lines decide how long a rebuild takes?** When the input of a step changes, Docker re-runs that step and every step after it. Copying `package.json` and the lockfile first and running `npm ci` before copying the code keeps the slow install cached when only code changes. In build d, a code change came before the slow step and forced it to run again.

3. **Why did `docker stop` take longer before the SIGTERM handler?** Node was PID 1 with no SIGTERM handler, so the signal was ignored. Docker waited for its timeout and then sent SIGKILL (exit code 137). With the handler, the app closes the server and exits with 0 in under a second.

4. **Three things the naive image contained that `course-api:lab2` does not:** the `.env` file with the secret, development dependencies such as nodemon, and the Dockerfile and README files. It also used the full 1.66 GB Debian `node:24` image and ran as root.