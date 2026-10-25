# Lab 3 · Branch, review, release

<!-- Copy this file to lab3/README.md (Lab 3, step 1.4) and fill it in as you go.
     Keep the headings exactly as they are: the grader looks for them by name.
     Put command output in the ```text blocks. -->

- **Fork:** https://github.com/Chamalka001/turiba-devops-labs
- **Release:** https://github.com/Chamalka001/turiba-devops-labs/releases/tag/v0.1.0
- **Reviewer:** @sachidumaleesha

## Part 1 · Set up

Output of `ssh -T git@github.com` (or `gh auth status`):

```
$ ssh -T git@github.com
Hi Chamalka001! You've successfully authenticated, but GitHub does not provide shell access.

```

Output of `git remote -v`:

```
$ git remote -v
origin  git@github.com:Chamalka001/turiba-devops-labs.git (fetch)
origin  git@github.com:Chamalka001/turiba-devops-labs.git (push)
upstream        https://github.com/mleitass/turiba-devops-labs.git (fetch)
upstream        https://github.com/mleitass/turiba-devops-labs.git (push)

```

## Part 2 · Playground

The hash in `.git/refs/heads/main` (step 2.3):

c5e81114cb73e318a67a94956fe26c79afefa202

The first line of `git cat-file -p HEAD` (step 2.3):

tree 6246df39db4c0193d1ee086b4532c335d9cc4ad0

The conflict block you saw in `config/app.env` (step 2.6):

```
<<<<<<< HEAD
REPLICAS=2
=======
REPLICAS=3
>>>>>>> feature/scale-up
LOG_LEVEL=info

```

Which value you kept, and why:

I kept REPLICAS=2, because it is safer to start with fewer copies and add more later when we see that we need them.

The final `git log --oneline --graph --all` (step 2.6):

```
*   ed15d64 (HEAD -> main) Merge branch 'feature/scale-up'
|\
| * 7b129d9 (feature/scale-up) feat: scale to 3 replicas
* | aef1c28 feat: scale to 2 replicas
|/
*   4daa0f3 Merge branch 'feature/logging'
|\
| * e1091d0 (feature/logging) feat: add log level
* | 3752b24 docs: add README
|/
* bfd3339 (feature/healthcheck) feat: add health message
* c5e8111 feat: print done after deploy
* 8919dd4 feat: add app config
* 842f861 feat: add deploy script

```

## Part 3 · Rescue missions

### m1 (worked example)

- PASS line: `PASS m1 72048770`
- Symptom: deploy.sh was changed and broken (a typo, `--scale api=0`, `sleep 99999`). Nothing was staged or committed.
- Command: `git restore deploy.sh`
- Why this command, and not a more dangerous one: it puts back only this one file from the last commit. `git reset --hard` would also throw away every other uncommitted change, and deleting the folder would lose the history too. The broken edit is gone for good, because it was never committed.

### m2

- PASS line: `PASS m2 f3cdde8c`
- Symptom: `.env` with a database password was staged together with a good change in `app.js`.
- Command: `git restore --staged .env`, then `echo ".env" >> .gitignore`, `git add .gitignore`, `git commit -m "feat: read port from PORT env var"`
- Why this command, and not a more dangerous one: it takes only `.env` off the staging area and keeps the good `app.js` change. `git reset --hard` would also throw away my real work. The password never went into a commit, because a password in history stays there and anyone with the repo can read it.

### m3

- PASS line: `PASS m3 2679427e`
- Symptom: the last commit had a bad message (`fxi typo`) and CONTRIBUTING.md was missing from it. Nothing was pushed.
- Command: `git add CONTRIBUTING.md`, then `git commit --amend -m "docs: fix typos in README"`
- Why this command, and not a more dangerous one: `--amend` replaces only the last commit and keeps the README fix. `git reset --hard HEAD~1` would delete the commit and the README fix with it. `--amend` is safe only because nothing was pushed yet.

### m4

- PASS line: `PASS m4 153637ad`
- Symptom: two commits were on main, but they belong on a branch called feature/cache. Nothing was pushed.
- Command: `git branch feature/cache`, then `git reset --hard origin/main`
- Why this command, and not a more dangerous one: I made the new branch first, so both commits stayed safe with a name. If I had run `git reset --hard` first, the commits would have lost their name and I would need `git reflog` to find them. The reset was safe because my working tree was clean and nothing was pushed.

### m5

- PASS line: `PASS m5 7ab29ac2`
- Symptom: the commit `feat: set replicas to 0` was already pushed, and my teammate already had it.
- Command: `git revert aa658f3 --no-edit`, then `git push`
- Why this command, and not a more dangerous one: `git revert` adds a new commit that cancels the bad one, and it keeps history. `git reset --hard` plus a force push would erase the commit on the server, but my teammate still has it. Their history would then be different from the server, and their next pull or push could fail or bring the bad commit back.

### m6

- PASS line: `PASS m6 2f102d61`
- Symptom: three commits were gone from `git log` after `git reset --hard HEAD~3`.
- Command: `git reflog` to find the old commit, then `git reset --hard 9f87f4b`
- Why this command, and not a more dangerous one: the reflog still remembers where HEAD was before the bad reset, so I could go back to that exact commit. I used the commit ID and not `HEAD@{1}`, because the number in braces changes after every move. It was safe because my working tree was clean. Git deletes unused commits after some weeks, so I should rescue early.

## Part 4 · Protected main and pull requests

The rejected push to `main` (step 4.2):

```text
$ git push
remote: error: GH013: Repository rule violations found for refs/heads/main.
remote: Review all repository rules at https://github.com/Chamalka001/turiba-devops-labs/rules?ref=refs%2Fheads%2Fmain
remote:
remote: - Changes must be made through a pull request.
remote:
remote: - 4 of 4 required status checks are expected.
remote:
To github.com:Chamalka001/turiba-devops-labs.git
 ! [remote rejected] main -> main (push declined due to repository rule violations)
error: failed to push some refs to 'github.com:Chamalka001/turiba-devops-labs.git'
```

- PR #5 (`.gitignore`): https://github.com/Chamalka001/turiba-devops-labs/pull/5
- PR #6 (`/healthz` docs, with the conflict): https://github.com/Chamalka001/turiba-devops-labs/pull/6

## Part 5 · Release

- Release: https://github.com/Chamalka001/turiba-devops-labs/releases/tag/v0.1.0
- Why `0.1.0` and not `1.0.0`? In Semantic Versioning, a version that starts with 0 means the project is still early and things can change. `1.0.0` means the API is stable and ready for people to depend on. This is my first release of the course API, so `0.1.0` is the honest number.

## Answers

1. In mission 5, why `git revert` and not `git reset --hard` plus a force push?

Because the bad commit was already pushed, and my teammate already had it. `git revert` adds a new commit that cancels the bad one. History stays the same, so my teammate can just run `git pull`. `git reset --hard` plus a force push would erase the commit on the server only. My teammate's copy would be different from the server, and their next pull or push could fail, or bring the bad commit back. It can also destroy other people's work.

2. Why couldn't Git resolve the `CHANGELOG.md` conflict by itself, and how did you decide what to keep?

Both PRs added a new line at the same place, at the end of the `### Added` list. Git cannot know which line should come first, or if one of them should be dropped, so it put markers in the file and asked me. Both lines described real changes, so I kept both and deleted only the three marker lines (`<<<<<<<`, `=======`, `>>>>>>>`). The order did not matter. Then I ran `git diff --check` to be sure no markers were left.

3. A colleague says "I'll make a PR to get the latest code." What do they mean, and which command is it?

They mix up two things. A pull request (PR) does not download code. It asks other people to merge my branch into another branch. To get the latest code, the command is `git pull` (it is `git fetch` and `git merge` together). In this lab, `main` is protected, so I get course updates in a different way: `git fetch upstream`, then `git switch -c chore/sync-upstream`, then `git merge upstream/main`, then push the branch and open a PR in my fork. I merge that PR with a merge commit.

4. After `1.0.0`: what is the next version for (a) a bug fix, (b) a new `/api/todos/search` endpoint, (c) renaming `/api/todos` to `/api/tasks`?

(a) A bug fix is `1.0.1` (patch). (b) A new endpoint that does not break anything is `1.1.0` (minor). (c) Renaming `/api/todos` to `/api/tasks` breaks the people who use the old address, so it is `2.0.0` (major).
