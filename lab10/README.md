# Lab 10 · Build it from code with OpenTofu

GitHub user: Chamalka001 · Fallback chart used: no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores). Paste command output inside the ```text blocks. Never paste the password, the state file or the kubeconfig. -->

## 1 · The cluster and the app from code

```text
$ tofu plan                                    (Part 1: the end of the first plan)
Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + kube_context = (known after apply)

$ time tofu apply -auto-approve                (Part 2: from zero, the last lines and the time)
# first try: the Helm release timed out because the postgres:17-alpine image downloaded slowly
│ Error: Helm release error
│ context deadline exceeded
real    5m42.492s
# I did: helm uninstall course-app -n course, then tofu apply again
helm_release.course_app: Creation complete after 20s [id=course-app]
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
Outputs:
kube_context = "kind-course"
real    0m23.551s

$ tofu plan                                    (right after that apply)
No changes. Your infrastructure matches the configuration.

$ grep -c '"result"' terraform.tfstate
2
```

- 🔮 Before the first plan (Part 1): how many resources will it add, and which values will be `known after apply`? I predicted 1 resource (the kind cluster), and that its address, certificates, kubeconfig and the output `kube_context` would be `known after apply`, because these values only exist after the cluster is built. What happened: `Plan: 1 to add, 0 to change, 0 to destroy.` and all of those values were `(known after apply)`. It matched.
- 🔮 Before the second plan (Part 2): what will `tofu plan` say right after `tofu apply`? I predicted `No changes. Your infrastructure matches the configuration.` because the code, the state and the cluster all agree. What happened: the plan said exactly that. One difference: my first apply from zero hit the 5 minute Helm timeout, because my internet is slow and the postgres image took about 11 minutes to download. The Pods were fine after the download, so I ran `helm uninstall course-app -n course` and `tofu apply` again. That second apply took 23 seconds because the image was already on the node.
- 🔮 Where is the database password now? I predicted it is in `terraform.tfstate` in plain text, because the state keeps every value, even the ones that plans hide as `(sensitive value)`. What happened: `grep -c '"result"' terraform.tfstate` gave 2, and `tofu state show kubernetes_secret_v1.db` only shows `data = (sensitive value)`. So the screen hides the password, but the state file still has it. That is why the state never goes into Git.

## 2 · Change, drift, rebuild

```text
$ tofu apply -var api_replicas=2               (the plan: the symbol and the value that changes)
  ~ update in-place (current -> planned)
  # helm_release.course_app will be updated in-place
      ~ set = [
          ~ {
                name  = "api.replicas"
              ~ value = "3" -> "2"
            },
        ]
Plan: 0 to add, 1 to change, 0 to destroy.
Apply complete! Resources: 0 added, 1 changed, 0 destroyed.

$ helm history course-app -n course
REVISION        UPDATED                         STATUS          CHART               APP VERSION     DESCRIPTION
1               Fri Nov 20 02:06:01 2026        superseded      course-app-0.1.0    0.2.0           Install complete
2               Fri Nov 20 02:08:11 2026        deployed        course-app-0.1.0    0.2.0           Upgrade complete

$ kubectl -n course get deployment api
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
api    2/2     2            2           2m15s

$ tofu plan -var api_replicas=2                (after kubectl -n course delete secret db-secret)
  # kubernetes_secret_v1.db will be created
  + resource "kubernetes_secret_v1" "db" {
Plan: 1 to add, 0 to change, 0 to destroy.

$ tofu plan -var api_replicas=2                (after kubectl -n course scale deployment api --replicas=1)
  # kubernetes_secret_v1.db will be created
  + resource "kubernetes_secret_v1" "db" {
Plan: 1 to add, 0 to change, 0 to destroy.

$ tofu apply                                   (the plan, and how many API Pods run afterwards)
  + create
  ~ update in-place (current -> planned)
      ~ value = "2" -> "3"
Plan: 1 to add, 1 to change, 0 to destroy.
Apply complete! Resources: 1 added, 1 changed, 0 destroyed.
$ kubectl -n course get deployment api
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
api    3/3     3            3           3m35s

$ time tofu destroy -auto-approve / time tofu apply -auto-approve      (the last lines and the times)
Destroy complete! Resources: 5 destroyed.
real    0m13.595s
# apply from zero, first try: Helm timeout while postgres downloaded
│ Error: Helm release error
│ context deadline exceeded
real    5m42.492s
# helm uninstall course-app -n course, then tofu apply again
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
real    0m23.551s
$ tofu plan
No changes. Your infrastructure matches the configuration.

$ curl -s localhost:8080/api/todos             (after the rebuild)
[]
```

- 🔮 Step 1, `-var api_replicas=2`: which symbol, and which value changes? I predicted `~ update in-place` with `~ value = "3" -> "2"` inside `set`, because the release stays and Helm upgrades it, so `helm history` would show revision 2. What happened: exactly that. `Plan: 0 to add, 1 to change, 0 to destroy.`, `helm history` shows revision 1 `superseded` and revision 2 `deployed`, and the `api` Deployment is `2/2`.
- 🔮 Step 2, the deleted Secret and the scaled Deployment: what will each plan show? I predicted `+ create` for `kubernetes_secret_v1.db` (`Plan: 1 to add`) after the Secret was deleted, because OpenTofu reads every resource it manages from the cluster before it plans. For the scaled Deployment I predicted the same plan and nothing more, because the Deployment is inside the Helm release, and OpenTofu only watches the release. What happened: both plans were the same, `Plan: 1 to add, 0 to change, 0 to destroy.` OpenTofu could not see the drift in the Deployment.
- 🔮 Step 3, a plain `tofu apply`: what will the plan contain, and how many API Pods afterwards? I predicted `+ create` for the Secret and `~ update in-place` for the release (`"2" -> "3"`), so `Plan: 1 to add, 1 to change, 0 to destroy.`, and 3 API Pods afterwards, because the Helm upgrade sets the Deployment back to 3 and also undoes the `kubectl scale`. What happened: exactly that, and `api` shows `3/3`.
- 🔮 Step 4, after `tofu destroy` and `tofu apply`: is the todo still there? I predicted no (`[]`), because the database lived on a volume inside the cluster, and `tofu destroy` deleted the cluster. What happened: `curl` gave `[]`. The code came back, but the data did not, because data is not code. One difference in time: the first apply from zero hit the 5 minute Helm timeout (slow postgres image download on my internet), so I did `helm uninstall course-app -n course` and applied again, which took 23 seconds.

## 3 · What broke and how I found it

- 🔮 Before `tofu plan` on the teammate's branch, I expected this to go wrong first: the provider stage, with an error about the lock file or the providers, because OpenTofu works in stages (providers and lock file first, then the code and chart, then the cluster) and the teammate touched `required_providers`. What happened: `Error: Inconsistent dependency lock file`, as expected.

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | `tofu plan` stopped at the lock file stage. The provider source in `versions.tf` was changed from `tehcyx/kind` to `hashicorp/kind`, which does not exist. | `cd infra && tofu plan`: `provider registry.opentofu.org/hashicorp/kind: required by this configuration but no version is selected` | `fix(infra): use the tehcyx/kind provider source again` (a68034f) |
| 2 | After fault 1, `tofu plan` could not find the chart. The path was `../chart/course-app`, without the s. | `tofu plan`: `Unable to locate chart ./../chart/course-app: path "./../chart/course-app" not found` | `fix(infra): point the chart path back to charts/course-app` (39961f3) |
| 3 | The plan worked but wanted to replace the Helm release, because the namespace was hard-coded to `"courses"`. Applying it would have deleted my release and the database, and then failed. I did not apply it. | `tofu plan`: `~ namespace = "course" -> "courses" # forces replacement` and `-/+ destroy and then create replacement`, `Plan: 1 to add, 0 to change, 1 to destroy.` | `fix(infra): put the release back in the course namespace` (b95cd52) |

## 4 · In my own words

- Why `.terraform.lock.hcl` goes into Git and `terraform.tfstate` doesn't: The lock file records the exact provider versions, so everyone who clones the repo gets the same providers. It has no secrets. The state file holds every value in plain text, including the database password, and the real cluster state, so it must stay out of Git. It is like a shopping list (fine to share) and a bank card (never share).
- What `-/+ … # forces replacement` means, and what it would have cost me in Part 4: It means OpenTofu cannot change that setting in place, so it must destroy the resource and create a new one. For the Helm release this meant uninstall and then install. In Part 4 it would have deleted my release and the database volume with my data, and then failed because the namespace `courses` did not exist.
- What I would lose by ending the day with `kind delete cluster` instead of `tofu destroy`: OpenTofu's state would still say the cluster exists, but it would be gone. Then `plan` and `apply` give errors like `could not locate any control plane nodes`, and I would have to delete the state files by hand and start again. `tofu destroy` removes everything in the right order and keeps the state honest.
