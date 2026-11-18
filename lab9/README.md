# Lab 9 · Package it with Helm

GitHub user: Chamalka001 · Fallback image used: no · Fallback Mission 8 files used: no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores). Paste command output inside the ```text blocks. Never paste your password. -->

## 1 · The chart

```text
$ helm lint charts/course-app
==> Linting charts/course-app
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed

$ helm template course-app charts/course-app -n course --set api.replicas=2 | grep -E 'replicas:|image:'
  replicas: 2
          image: postgres:17-alpine
          image: "ghcr.io/chamalka001/turiba-devops-labs/api:0.2.0"
  replicas: 1
          image: postgres:17-alpine
```

- 🔮 Before running `helm template … --set api.replicas=2`: which lines of the output change, and which file decides that? I predicted only the `replicas:` line of the API Deployment would change, from 3 to 2, because the template reads `.Values.api.replicas`. Without `--set`, the file `values.yaml` decides the number (3). What happened: the API Deployment showed `replicas: 2`. The API image line and the PostgreSQL Deployment (`replicas: 1`) did not change. Without `--set` I got `replicas: 3`. My prediction was correct.
- In my own words, why the Namespace and the Secret are **not** in the chart: The Namespace must exist before `helm install -n course`, so I create it with `kubectl`. If the chart also made it, Helm would refuse, because Helm never takes over an object it did not make. The Secret holds the database password. If it was a file in the chart, the password would be in Git. So I create it with a command, outside the chart.

## 2 · Install, upgrade, rollback

```text
$ helm install course-app charts/course-app -n course
NAME: course-app
LAST DEPLOYED: Tue Nov 17 22:19:36 2026
NAMESPACE: course
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None

$ kubectl -n course get pods
NAME                   READY   STATUS    RESTARTS   AGE
api-544cbfc77b-mjk7h   1/1     Running   0          36s
api-544cbfc77b-qkmwc   1/1     Running   0          36s
api-544cbfc77b-x96h5   1/1     Running   0          36s
db-864878fdcf-wchwx    1/1     Running   0          36s

$ curl -s -X POST -H 'Content-Type: application/json' -d '{"title":"packaged by helm"}' localhost:8081/api/todos
{"id":1,"title":"packaged by helm","done":false,"created_at":"2026-11-17T17:21:34.759Z"}

$ helm upgrade course-app charts/course-app -n course --set api.image.tag=does-not-exist
Release "course-app" has been upgraded. Happy Helming!
NAME: course-app
LAST DEPLOYED: Tue Nov 17 22:52:07 2026
NAMESPACE: course
STATUS: deployed
REVISION: 2
DESCRIPTION: Upgrade complete
TEST SUITE: None

$ kubectl -n course get pods -l app=api        (about 30 seconds later)
NAME                   READY   STATUS         RESTARTS   AGE
api-544cbfc77b-mjk7h   1/1     Running        0          5m1s
api-544cbfc77b-qkmwc   1/1     Running        0          5m1s
api-544cbfc77b-x96h5   1/1     Running        0          5m1s
api-5586f84d89-zzfhm   0/1     ErrImagePull   0          34s

$ curl -s localhost:8081/api/todos
[{"id":1,"title":"packaged by helm","done":false,"created_at":"2026-11-17T17:21:34.759Z"}]

$ helm history course-app -n course            (before the rollback)
REVISION   UPDATED                    STATUS       CHART              APP VERSION   DESCRIPTION
1          Tue Nov 17 22:19:36 2026   superseded   course-app-0.1.0   0.2.0         Install complete
2          Tue Nov 17 22:52:07 2026   deployed     course-app-0.1.0   0.2.0         Upgrade complete

$ helm rollback course-app 1 -n course
Rollback was a success! Happy Helming!

$ kubectl -n course rollout status deployment/api
deployment "api" successfully rolled out

$ helm history course-app -n course            (after the rollback)
REVISION   UPDATED                    STATUS       CHART              APP VERSION   DESCRIPTION
1          Tue Nov 17 22:19:36 2026   superseded   course-app-0.1.0   0.2.0         Install complete
2          Tue Nov 17 22:52:07 2026   superseded   course-app-0.1.0   0.2.0         Upgrade complete
3          Tue Nov 17 22:53:22 2026   deployed     course-app-0.1.0   0.2.0         Rollback to 1

$ helm get values course-app -n course
USER-SUPPLIED VALUES:
null

$ helm uninstall course-app -n course
release "course-app" uninstalled

$ kubectl -n course get pvc,secret
NAME                            STATUS        VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/db-data   Terminating   pvc-086b15a5-501e-494d-a2dc-2c9efbe3054e   1Gi        RWO            standard       <unset>                 34m

NAME               TYPE     DATA   AGE
secret/db-secret   Opaque   1      34m

$ helm install course-app charts/course-app -n course
NAME: course-app
LAST DEPLOYED: Tue Nov 17 22:53:51 2026
NAMESPACE: course
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None

$ kubectl -n course get pods
NAME                   READY   STATUS    RESTARTS   AGE
api-544cbfc77b-8ttq2   1/1     Running   0          19s
api-544cbfc77b-l54f8   1/1     Running   0          19s
api-544cbfc77b-rh42k   1/1     Running   0          19s
db-864878fdcf-srmn6    1/1     Running   0          19s

$ helm list -n course
NAME         NAMESPACE   REVISION   UPDATED                                     STATUS     CHART              APP VERSION
course-app   course      1          2026-11-17 22:53:51.559041311 +0530 +0530   deployed   course-app-0.1.0   0.2.0
```

- 🔮 Before the upgrade with a tag that doesn't exist: will `helm upgrade` say it failed? Will `GET /api/todos` still answer? I predicted Helm would say the upgrade worked, but the new Pod would not start. The API would still answer, because Helm does not wait for the new Pods, and the rolling update keeps the old Pods running. What happened: Helm said `STATUS: deployed`, revision 2. The new Pod showed `ErrImagePull`. The three old Pods stayed `Running`, and `GET /api/todos` still returned my todo. My prediction was correct.
- 🔮 Before the rollback: which revision number will `helm rollback course-app 1` create? I predicted revision 3, because a rollback does not rewrite history. It makes a new revision with the settings of revision 1. What happened: `helm history` showed revision 3 with `Rollback to 1`. Revisions 1 and 2 became `superseded`. `helm get values` showed `null`, so the bad tag was gone. My prediction was correct.
- 🔮 Before `helm uninstall` and a new `helm install`: will your todo still be there? I predicted no, because the PVC `db-data` is part of the release, so `helm uninstall` deletes it together with the data. What happened: `kubectl get pvc` showed `db-data` as `Terminating`, the Secret stayed, and after the new install the list was empty. My prediction was correct.

```text
$ curl -s localhost:8081/api/todos             (after uninstall and install)
[]
```

Notes about my own run (a mismatch from the lab text, explained):

- I used port 8081 for the port-forward and not 8080, because port 8080 was already used by my Docker Compose web container.
- At the start, the kind cluster could not pull `postgres:17-alpine` and my API image, so the Pods were stuck in `Init:0/1`. I saved both images with `docker save --platform linux/amd64`, loaded them with `kind load image-archive`, and deleted the Pods. Then all 3 API Pods and the database were `1/1 Running`. The chart itself was not changed for this.

## 3 · What broke and how I found it

- 🔮 Before upgrading with the teammate's branch, I expected this to go wrong first: Helm would fail while it renders the templates, because the image repository was read with a wrong key. What happened: this was correct. `helm lint` and `helm upgrade` both stopped with `nil pointer evaluating interface {}.repository`. After I fixed it, the next problem came (the Namespace), and then a silent one (the replicas).

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | `helm lint` failed and `helm upgrade` failed before anything reached the cluster. The template read `images` instead of `image`, and `values.yaml` has no `images`. | `helm lint charts/course-app` and `helm upgrade course-app charts/course-app -n course`. Line: `executing "course-app/templates/api-deployment.yaml" at <.Values.api.images.repository>: nil pointer evaluating interface {}.repository` | `47f9861` fix(chart): read the image repository from api.image |
| 2 | `helm upgrade` failed because the teammate added `templates/namespace.yaml`. The Namespace `course` was made by me with `kubectl`, so Helm does not own it. | `helm upgrade course-app charts/course-app -n course`. Line: `Namespace "course" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata` | `9d441e6` fix(chart): remove namespace from chart |
| 3 | `helm upgrade` said `deployed`, but `kubectl get deployment api` showed only 1/1. In `values.yaml`, `replicas` had become `replicaCount`, so the template got an empty value and Kubernetes used its default (1). `helm lint` did not show it. | `helm template course-app charts/course-app -n course \| grep -n -A2 "replicas:"`. Line: `60:  replicas:` (empty). Then `kubectl -n course get deployment api`: `api   1/1` | `c421784` fix(chart): restore api replicas value. After the fix: `api   3/3` |
