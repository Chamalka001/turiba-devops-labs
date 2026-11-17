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

- 🔮 Before running `helm template … --set api.replicas=2`: which lines of the output change, and which file decides that? I predicted the API Deployment replicas line would change from 3 to 2 because the api.replicas value in values.yaml is used by the API Deployment template. What happened: the API Deployment rendered with 2 replicas, while the image lines stayed the same. The PostgreSQL Deployment still had 1 replica.

- In my own words, why the Namespace and the Secret are **not** in the chart: The Namespace is created separately before installing the chart, so Helm does not try to manage an existing Namespace. The Secret is also created separately because it contains sensitive information such as the database password. Keeping it outside the chart avoids storing sensitive information in the chart or GitHub repository.

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
____

$ helm upgrade course-app charts/course-app -n course --set api.image.tag=does-not-exist
Release "course-app" has been upgraded. Happy Helming!
NAME: course-app
LAST DEPLOYED: Tue Nov 17 22:52:07 2026
NAMESPACE: course
STATUS: deployed
REVISION: 2
DESCRIPTION: Upgrade complete
TEST SUITE: None
____

$ kubectl -n course get pods -l app=api        (about 30 seconds later)
NAME                   READY   STATUS         RESTARTS   AGE
api-544cbfc77b-mjk7h   1/1     Running        0          5m1s
api-544cbfc77b-qkmwc   1/1     Running        0          5m1s
api-544cbfc77b-x96h5   1/1     Running        0          5m1s
api-5586f84d89-zzfhm   0/1     ErrImagePull    0          34s

____

$ helm history course-app -n course            (before the rollback)
REVISION        UPDATED                         STATUS    CHART                   APP VERSION     DESCRIPTION
1               Tue Nov 17 22:19:36 2026        superseded course-app-0.1.0        0.2.0           Install complete
2               Tue Nov 17 22:52:07 2026        deployed   course-app-0.1.0        0.2.0           Upgrade complete
____

$ helm rollback course-app 1 -n course
Rollback was a success! Happy Helming!
____

$ helm history course-app -n course            (after the rollback)
REVISION        UPDATED                         STATUS          CHART               APP VERSION     DESCRIPTION
1               Tue Nov 17 22:19:36 2026        superseded      course-app-0.1.0    0.2.0           Install complete
2               Tue Nov 17 22:52:07 2026        superseded      course-app-0.1.0    0.2.0           Upgrade complete
3               Tue Nov 17 22:53:22 2026        deployed        course-app-0.1.0    0.2.0           Rollback to 1
____
```

- 🔮 Before the upgrade with a tag that doesn't exist: will helm upgrade say it failed? Will GET /api/todos still answer? I predicted Helm would report the upgrade as successful, but the new API pod would fail to pull its image. The existing API pods would still answer requests because the upgrade command does not wait for the new pods to become ready, and the old pods can continue running. What happened: Helm reported the upgrade as deployed with revision 2. The new pod showed ErrImagePull, while the three old API pods remained Running and the GET request still returned the saved todo.
- 🔮 Before the rollback: which revision number will helm rollback course-app 1 create? I predicted revision 3 because a rollback creates a new revision using the configuration of the selected previous revision. What happened: Helm created revision 3, with the description Rollback to 1.

- 🔮 Before helm uninstall and a new helm install: will your todo still be there? I predicted no because the database PVC is managed by the Helm release and is deleted when the release is uninstalled. What happened: the PVC was deleted during uninstall, and after reinstalling the application the todo list was empty.

```text
$ curl -s localhost:8080/api/todos             (after uninstall and install)
[]
____
```

## 3 · What broke and how I found it

- 🔮 Before upgrading with the teammate's branch, I expected this to go wrong first: the API image repository value would be referenced with the wrong key, causing Helm to fail while rendering the chart.

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | 

The API image repository used images instead of image, so Helm could not find the repository value. | helm lint charts/course-app — nil pointer evaluating interface {}.repository at .Values.api.images.repository. The upgrade also failed with the same error. | 47f9861 — changed .Values.api.images.repository to .Values.api.image.repository. |
| 2 | The chart included a Namespace template even though the course Namespace already existed and was not managed by Helm. | helm upgrade course-app charts/course-app -n course — Namespace "course" ... exists and cannot be imported into the current release: invalid ownership metadata. | 9d441e6 — removed charts/course-app/templates/namespace.yaml.|
| 3 | 

The API Deployment rendered without a replica count because the values file used replicaCount instead of replicas. Kubernetes created only one API replica. | helm template course-app charts/course-app -n course — replicas: was empty. kubectl -n course get deployment api showed 1/1. | c421784 — changed replicaCount back to replicas in values.yaml. The API Deployment then reached 3/3. |
