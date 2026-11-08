# Lab 7 · Run it on a cluster

GitHub user: Chamalka001 · Fallback image used: no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores). Paste command output inside the ```text blocks. -->

## 1 · Deploy

- 🔮 Before my first `apply`: how many Pods, and what their names start with? I predicted 3 Pods, with names starting with api-, because replicas is 3 and the Deployment is named api. What happened: I saw 3 Pods and their names start with api-. My prediction was right.

```text
$ kubectl -n course get deploy,rs,pods
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/api   3/3     3            3           3m56s

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/api-7b76f4dc84   3         3         3       3m56s

NAME                       READY   STATUS    RESTARTS   AGE
pod/api-7b76f4dc84-4mfpr   1/1     Running   0          3m56s
pod/api-7b76f4dc84-mgqcd   1/1     Running   0          3m56s
pod/api-7b76f4dc84-q54cx   1/1     Running   0          3m56s

$ curl -s localhost:8081/
{"name":"course-api","version":"0.1.0"}
```

## 2 · Heal and scale

- 🔮 After deleting one Pod, two seconds later I expected to see 3 Pods again, with one new Pod with a new name and a small age, because the Deployment always keeps replicas: 3. What happened: the new Pod api-7b76f4dc84-jl7h5 was already Running with age 2s, and the deleted Pod was already gone. My prediction was right.

```text
$ kubectl -n course get pods        (after the delete)
NAME                   READY   STATUS    RESTARTS   AGE
api-7b76f4dc84-jl7h5   1/1     Running   0          2s
api-7b76f4dc84-mgqcd   1/1     Running   0          10m
api-7b76f4dc84-q54cx   1/1     Running   0          10m

$ kubectl -n course get pods        (with replicas: 5)
NAME                   READY   STATUS    RESTARTS   AGE
api-7b76f4dc84-dkzt9   1/1     Running   0          2s
api-7b76f4dc84-jl7h5   1/1     Running   0          28s
api-7b76f4dc84-mgqcd   1/1     Running   0          11m
api-7b76f4dc84-p9nn4   1/1     Running   0          2s
api-7b76f4dc84-q54cx   1/1     Running   0          11m
```

## 3 · Roll and roll back

- 3a: I rolled from the tag 0.2.0 to the tag 7e4f231f5f5b261856f7b42693b5e8a78e9a686e.

```text
$ kubectl -n course rollout status deployment/api
Waiting for deployment "api" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "api" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "api" rollout to finish: 1 old replicas are pending termination...
deployment "api" successfully rolled out
```

- 3b: 🔮 With the tag 9.9.9, I predicted that 3 old Pods keep running, because the new Pod cannot pull its image, so it never becomes ready, and a rolling update only removes an old Pod when a new one is ready. What happened: 3 old Pods stayed Running and the new Pod api-88f8f59df-plhrs went to ImagePullBackOff. My prediction was right.

```text
$ kubectl -n course get pods        (during the stall)
NAME                   READY   STATUS             RESTARTS   AGE
api-6bfcf9fd4f-8kktj   1/1     Running            0          12m
api-6bfcf9fd4f-9vt5w   1/1     Running            0          12m
api-6bfcf9fd4f-p7vdg   1/1     Running            0          12m
api-88f8f59df-plhrs    0/1     ImagePullBackOff   0          87s

$ kubectl -n course rollout undo deployment/api
Warning: resource deployments/api was previously managed with 'kubectl apply'. Rolling back will not update the kubectl.kubernetes.io/last-applied-configuration annotation, which may cause unexpected behavior on future 'kubectl apply' operations. Consider using 'kubectl apply' with your previous configuration file instead.
deployment.apps/api rolled back
```

- What the `rollout undo` warning means, in my own words, and what I did so that Git and the cluster agree again: rollout undo changes only the cluster, not my file in Git, so the cluster and the file disagree (the file still said 9.9.9). I ran git restore k8s/ and then kubectl apply -f k8s/, and after that git diff was empty and rollout status said the rollout was done.

## 4 · What broke and how I found it

- 🔮 Before applying the teammate's branch, I expected this to go wrong first: ____

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | ____ | ____ | ____ |
| 2 | ____ | ____ | ____ |
| 3 | ____ | ____ | ____ |
