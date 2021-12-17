# Useful commands for CKA Exam

## Setup your bash
```
alias k=kubectl
complete -F __start_kubectl k
export do="--dry-run=client -o yaml" # k get po $do
export now="--force --grace-period 0" # k delete po $now
source <(kubectl completion bash)
source <(helm completion bash)
source <(linkerd completion bash)
```

Setup your vim

```
set ts=2
set expandtab
set shiftwidth=2
```
## Command shortcuts

### Extract name from context-line

Erase beginning of a string in a line to retain just the name
E.g.: in `current-context: context01`, to only extract the name of the context,

`cat ~/.kube/config | grep current-context | sed -e "s/current-context: //"`

### Poll for something periodically

Run `date` command every one second
```
while true; 
do date >> /vol/date.log; 
sleep 1; 
done
```

## Kubeconfig commands

Using kubectl
```
k config view -o yaml
k config view -o jsonpath="{.contexts[*].name}"
k config view -o jsonpath="{.contexts[*].name}" | tr " " "\n" # new lines
```
Get current context without using kubectl (parse .kube/config)

```
`cat ~/.kube/config | grep current-context | sed -e "s/current-context: //"` > file.txt
```

## Pod Manipulations

### Extract a YAML without creating the pod
`k run pod1 --image=nginx $do > pod1.yaml`

### Create a busybox pod and run meaningful commands (not just sleep)
```
- image: busybox:1.31.1
  name: busyboxc
  command: ["sh", "-c", "tail -f /vol/date.log"]
```

### Deploy a pod on named node without using labels
Add `nodeName: cluster1-master1` at the same level as containers. This is useful when the question specifically asks not to use labels on the nodes.

[nodeName](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodename)

[OPTIONAL] You might need to add tolerations to the pod to deploy on master node
```
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
    effect: "NoSchedule"
```

[Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

### Deploy a pod on named node using labels

Step 1: add label to the node that you are targeting
`kubectl label nodes <node-name> <label-key>=<label-value>`
E.g.:
`kubectl label nodes master type=master` if the question is asking for deployment on master node
`kubectl label nodes master type=master` - if pod is to be scheduled on worker node

Step 2: check for taints on the node (especially master node)
`kubectl describe nodes master | grep Taint`

Remove Taint (READ THE QUESTION WHETHER THIS IS ALLOWED)
`k taint nodes master node-role.kubernetes.io/master:NoSchedule-` (note the `-` at the end - that is remove)

Step 3: make changes to your pod manifest to include `nodeSelector`

```
spec:
  containers:
  - image: nginx
    name: pod1
    resources: {}
  nodeSelector:
    type: worker
```
