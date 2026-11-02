# Lab 6 · Test what you ship, ship a release

GitHub user: Chamalka001 · Reference pipeline used (Part 0 fallback): no

## 1 · Smoke test

- The names of my smoke-test steps in the `image` job: "Build for the smoke test (amd64, kept on the runner)" and "Smoke test the image"
- What my smoke test asks, and what makes it fail: It starts the image with `docker run` and asks `GET /` with curl (it retries 10 times). It fails if the container does not answer. Then it prints the container log with `docker logs` and exits with an error.
- 🔮 Part 1, before my first run: will my code pass the smoke test? I predicted it would pass because the code is the same as in Lab 5 and `GET /` does not need a database. What happened: It passed. The first curl try failed with "Connection reset by peer" because the API was still starting. The retry worked and it printed `{"name":"course-api","version":"0.1.0"}`.
- 🔮 Part 2, before the checks finished: which checks will go red after `break.sh`? I predicted only `image` would go red, because `lint`, `unit` and `integration` run the source code on the runner where every file exists, but the image only has the files that the Docker build gives it.

## 2 · What broke and how I found it

- The check and the step that went red: The `image` check, in the step "Smoke test the image". `lint`, `unit` and `integration` were green.
- The log line that showed the cause (copied from the log):

```text
Error: Cannot find module './banner'
```

- The cause, in my own words: The teammate's commit added `banner.js` and `server.js` now needs it. But the same commit also added `banner.js` to `api/.dockerignore`. So Docker did not copy the file into the image, even though my Dockerfile uses `COPY . .`. The container started, could not find the file, and crashed. So curl got no answer.
- Why `lint`, `unit` and `integration` stayed green: They run the code from the repository on the runner, where `banner.js` exists. They never use the Docker image, so `.dockerignore` does not matter to them. Only the smoke test starts the real image.
- My fix, and the commit that made it: I deleted only the two `banner.js` lines (a comment and the file name) from `api/.dockerignore`. I did not delete the whole file, because then `.env` and `node_modules` would go into the image. Commit e8e8752 "fix(api): keep banner.js in the image". After it, all four checks were green and PR #10 was merged.

## 3 · Release

- Release page: https://github.com/Chamalka001/turiba-devops-labs/releases/tag/v0.2.0
- Image: `ghcr.io/chamalka001/turiba-devops-labs/api:0.2.0`, platforms (from `docker buildx imagetools inspect`): linux/amd64 and linux/arm64
- One thing in Part 3 that didn't work the first time, and how I fixed it: Nothing failed in Part 3. The release run was green the first time. The run was quick (about 1 minute), so I did not trust the green tick alone. I checked the Release page, ran `docker buildx imagetools inspect` and did a logged-out `docker pull` to be sure.
