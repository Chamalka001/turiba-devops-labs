# Lab 5 · MY first CI pipeline

GitHub user: chamalka001 · Reference files used (Lab 2 fallback): no

![ci](https://github.com/chamalka001/turiba-devops-labs/actions/workflows/ci.yml/badge.svg?branch=main)

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores), and your user name placeholder in the first line and in the badge URL: write your GitHub user name exactly as GitHub shows it. -->

## 1 · Graph predictions

Part 1, step 1.2: fill in this section **before** you run `lab05 graph`. A wrong prediction that you explain earns full marks.

My drawing of the `lab05 graph` jobs, before the first run (boxes and arrows in text are fine):

```text
graph-build ---> graph-test ---+
                               +---> graph-package ---> graph-report
graph-lint --------------------+
(graph-report also waits for graph-build, graph-lint and graph-test)
````

| # | Question | I predicted, and why | What the run showed |
|---|----------|----------------------|---------------------|
| P1 | Which jobs start together? | graph-build and graph-lint, because they are the only jobs with no needs: line, so nothing makes them wait | graph-build and graph-lint started together. All five jobs green. |
| P2 | Will `graph-package` find `build.txt`? | No, because every job runs on its own fresh machine and jobs do not share files. graph-test has a step that gets the file, graph-package does not | Correct. graph-test downloaded build.txt (it has a download-artifact step). graph-package printed "build.txt: not here: jobs don't share files" (no download step). |
| P3 | With `break = build`: which jobs run, which fail, which are skipped? | graph-build fails on purpose. graph-lint runs because it does not need graph-build. graph-test is skipped because it needs graph-build. graph-package is skipped because it needs graph-test, which was skipped. graph-report still runs because it has if: always(). | Correct. graph-build failed (red). graph-lint ran and passed. graph-test and graph-package were skipped (grey). graph-report still ran (green) because of if: always(). |

## 2 · Pipeline

- B5, before I filled it in: `localhost` or `db`? I predicted db because in my Lab 4 compose.yaml the API used db as the database name.
- My first red run in Part 2: there was none. All four checks (lint, unit, integration, image) were green at once.
- What I changed to make it green: nothing, it was green at once.
- My lint findings, and how I fixed them (fixed, not silenced): no findings. npm run lint printed no problems, so I did not need to change any code, and I did not use eslint-disable.
- Part 2 pull request (URL): https://github.com/Chamalka001/turiba-devops-labs/pull/2
- Part 3: will the PR run push an image? I predicted no, because the file says push only when the event is a push to main, and a PR is not that. What happened: correct. In the image job of the PR run, the step "Log in to GHCR (main only)" was skipped, and push was false, so the image was only built and not pushed.
- My image on GHCR, from the `main` run: `ghcr.io/chamalka001/turiba-devops-labs/api:36e9d4177703a2cebde324d79392405b203379a0`

## 3 · Red PR

- Red PR (URL): https://github.com/Chamalka001/turiba-devops-labs/pull/3
- The test that failed, with Expected and Received (copied): "DELETE with an id that is not a number answers 404" failed. Expected: 404, Received: 500. Tests: 1 failed, 5 passed, 6 total.
- The cause line from "On failure: the API log" (copied): DELETE /api/todos/abc failed: invalid input syntax for type integer: "abc"
- The merge box while it was red (the text, copied): Some checks were not successful. ci / integration failed, ci / image was skipped, and Merge pull request was greyed out, so I could not merge.
- Which commit did the red check test: your branch alone, or your branch merged with `main`? It tested my branch merged with main. A pull_request run checks the temporary merge commit that GitHub makes from the PR branch and main (slide 9). I know this from the run summary page and the slide.

## 4 · Answers

**A1.** In my break: build run, graph-build failed (red) on purpose. graph-lint ran and passed, because it does not need graph-build. graph-test and graph-package were skipped (grey), because they depend on graph-build, directly or through graph-test. graph-report still ran and was green. It ran because it has if: always(). Without a condition, a job runs only when everything it needs succeeded. always() means run whatever happened before. Job list from my run: graph-build failed, graph-lint succeeded, graph-test skipped, graph-package skipped, graph-report succeeded.

**A2.** Every job runs on its own fresh machine, and jobs do not share files. graph-test has a download-artifact step that gets build.txt. graph-package has no such step, so it did not find it. A download-artifact step in graph-package (for the file that graph-build uploads with upload-artifact) would make it find build.txt.

**A3.** Both are right because the steps run in different places. In Lab 4, the API ran inside a container on the Compose network, and the name db is the service name that only works there. In CI, the integration steps run on the runner machine itself, and the database service publishes port 5432 to it, so the database is at localhost:5432. If the job ran in a container (container: node:24-alpine), the steps would be on the same network as the service, so DB_HOST would be db (the service label).

**A4.** The required check ci / integration failed (Expected 404, Received 500), and image was skipped. The run tested the temporary merge commit that GitHub makes from my branch and main (slide 9). The ruleset stopped the merge, because a required check was not successful, so Merge pull request was greyed out. On my Part 2 PR the image job ran but did not push, because push is true only when the event is a push to main. On a PR, the login step is skipped and the image is only built for amd64.
