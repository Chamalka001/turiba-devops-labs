# Lab 8 · Run the whole app

GitHub user: Chamalka001 · Fallback image used: no · Fallback Mission 7 files used: no

## 1 · Database with storage

- 🔮 Before deleting the database Pod: will your todo still be there afterwards? I did not write down a prediction before this test. What I understand now: the todo stays, because the data is on the PVC db-data and not inside the Pod. A new Pod mounts the same PVC. What happened: after I deleted the db Pod, a new Pod started and the todo was still listed.
- Note: port 8080 on my laptop was already used by my old Lab 4 Docker Compose app. My first test talked to that app by mistake (old todos from October). I redid the test with the port-forward on 8081, and the list then had only my own todo.

```text
$ kubectl apply -f k8s/ && kubectl -n course get pvc        (right after apply)
NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
db-data   Pending                                      standard       <unset>                 0s

$ kubectl -n course get pods,pvc        (after the db Pod started)
NAME                       READY   STATUS    RESTARTS   AGE
pod/api-6bfcf9fd4f-65xrm   1/1     Running   0          6m20s
pod/api-6bfcf9fd4f-fz7hs   1/1     Running   0          6m20s
pod/api-6bfcf9fd4f-xvdlp   1/1     Running   0          6m20s
pod/db-84f4dd66bb-gpvn6    1/1     Running   0          2m31s

NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/db-data   Bound    pvc-13317536-99cf-4e5e-9a93-239b43561d75   1Gi        RWO            standard       <unset>                 2m31s

$ curl -s -X POST -H 'Content-Type: application/json' -d '{"title":"survive a restart"}' localhost:8081/api/todos
{"id":1,"title":"survive a restart","done":false,"created_at":"2026-11-13T18:06:26.480Z"}

$ kubectl -n course get pods -l app=db        (before the delete)
NAME                  READY   STATUS    RESTARTS   AGE
db-84f4dd66bb-5s6q7   1/1     Running   0          93s

$ kubectl -n course delete pod -l app=db
pod "db-84f4dd66bb-5s6q7" deleted from course namespace

$ kubectl -n course get pods -l app=db        (after the delete)
NAME                  READY   STATUS    RESTARTS   AGE
db-84f4dd66bb-5q2c7   1/1     Running   0          2s

$ curl -s localhost:8081/api/todos
[{"id":1,"title":"survive a restart","done":false,"created_at":"2026-11-13T18:06:26.480Z"}]
```

The PVC was Pending right after apply and Bound once the db Pod started. This is normal on kind: it creates the volume only when a Pod uses the claim.

## 2 · Connect the API

```text
$ kubectl -n course logs deployment/api -c wait-for-db
Found 3 pods, using pod/api-6486888cf5-vdz7d
db:5432 - accepting connections

$ kubectl -n course logs deployment/api -c api
Found 3 pods, using pod/api-6964668d77-lk8qd
course-api 0.1.0 starting (Node.js v24.22.0)
API listening on port 5000
Database ready
```

- 🔮 Before looking: will `kubectl -n course get secret db-secret -o yaml` show your password? I did not write down a prediction before looking. What I saw, in words (don't paste the value): the data field has a password key with a long encoded text, not my plain password. It is only base64, which is an encoding and not encryption, so anyone who can read the Secret can decode it. The Secret keeps the password out of Git, but it does not lock it.

## 3 · Health and resources

- 🔮 On my working copy only, I pointed the **liveness** probe of the API at `/nope` and applied it. Will those Pods get traffic? Will they be restarted? I did not write down a prediction before this test. What happened: the new Pods were restarted again and again (RESTARTS 2, then 3). A failing liveness probe makes Kubernetes kill the container. The Pods showed 1/1 only for short moments between the restarts, and the readiness probe failed with "connection refused" while the container was starting again.

```text
$ kubectl -n course get pods -l app=api       (about a minute later)
NAME                   READY   STATUS    RESTARTS      AGE
api-5bf54fd669-9fn4r   1/1     Running   2 (18s ago)   88s
api-5bf54fd669-g48j6   1/1     Running   2 (10s ago)   81s
api-5bf54fd669-w29zz   1/1     Running   2 (6s ago)    77s

$ kubectl -n course describe pod api-5bf54fd669-w29zz         (only the Events)
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  94s                default-scheduler  Successfully assigned course/api-5bf54fd669-w29zz to kind-control-plane
  Normal   Pulled     93s                kubelet            spec.initContainers{wait-for-db}: Container image "postgres:17-alpine" already present on machine and can be accessed by the pod
  Normal   Created    93s                kubelet            spec.initContainers{wait-for-db}: Container created
  Normal   Started    93s                kubelet            spec.initContainers{wait-for-db}: Container started
  Normal   Created    23s (x3 over 92s)  kubelet            spec.containers{api}: Container created
  Normal   Started    23s (x3 over 92s)  kubelet            spec.containers{api}: Container started
  Normal   Killing    23s (x2 over 53s)  kubelet            spec.containers{api}: Container api failed liveness probe, will be restarted
  Warning  Unhealthy  22s (x6 over 92s)  kubelet            spec.containers{api}: Readiness probe failed: Get "http://10.244.0.12:5000/healthz": dial tcp 10.244.0.12:5000: connect: connection refused
  Warning  Unhealthy  3s (x8 over 73s)   kubelet            spec.containers{api}: Liveness probe failed: HTTP probe failed with statuscode: 404
```

- How I made Git and the cluster agree again afterwards: I ran `git restore k8s/ && kubectl apply -f k8s/`. After that `git diff` was empty and the new Pods (api-7644745969-...) started with RESTARTS 0.

## 4 · What broke and how I found it

- 🔮 Before applying the teammate's branch, I expected this to go wrong first: I did not write down a prediction before applying. What I saw first: the new api Pod was stuck in Init:0/1, and the 3 old Pods from main kept serving. The rolling update protected the app.

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | New Pod stuck in Init:0/1, the init container waited for ever | `kubectl -n course logs api-554c694949-vjqgj -c wait-for-db` showed `database:5432 - no response`; `git show HEAD` showed DB_HOST changed from db to database in app-config.yaml | 6ee369a fix(k8s): set DB_HOST back to db, then kubectl apply and rollout restart deployment/api |
| 2 | New Pod in CreateContainerConfigError | `kubectl -n course describe pod api-d9b677bb5-w58sf` showed `Error: couldn't find key pasword in Secret course/db-secret` | 83fd659 fix(k8s): correct the Secret key name to password |
| 3 | New Pod 0/1 Running, never ready | `kubectl -n course describe pod api-64cd5cc48b-bfnfv` showed `Readiness probe failed: Get "http://10.244.0.18:5001/healthz": dial tcp 10.244.0.18:5001: connect: connection refused` | 8a6fb0e fix(k8s): point the readiness probe back to port 5000 |

After the 3 fixes all Pods were 1/1 Running, and POST and GET /api/todos answered through the port-forward. The PR was merged with a merge commit, so the teammate's commit and my 3 fix commits are all in main. The required checks stayed pending first, because GitHub had disabled Actions on my fork. I enabled Actions again, and the checks turned green.
