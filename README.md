# Useful commands for CKA Exam

## Setup your bash
```
alias k=kubectl
complete -F __start_kubectl k
export do="--dry-run=client -o yaml" # k get po $do
export now="--force --grace-period 0" # k delete po $now - very useful since you do not have to wait for termination
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
:fire: kubectl

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
### Most common error and remediation
`error: error validating "pod1.yaml": error validating data: ValidationError(Pod.spec.containers): invalid type for io.k8s.api.core.v1.PodSpec.containers: got "map", expected "array";`
You have missed `-` somewhere :)

## Cluster and nodes
### count master and worker nodes
`k get nodes | grep master | wc -l` - masters
`k get nodes | grep worker | wc -l` - workers

### Pod IP address and network plug-in
:fire: config files

Directories for network plugin installation: `/opt/cni/bin`
```
root@master:/opt/cni/bin# ls
bandwidth  dhcp      flannel      host-local  loopback  portmap  sbr     tuning  weave-ipam  weave-plugin-2.8.1
bridge     firewall  host-device  ipvlan      macvlan   ptp      static  vlan    weave-net
```
CNI configuation would be in: `/etc/cni/net.d` with a filename starting with `10-`.

Pod IP address range would be within description of the CNI pod.
For Weave:
`kubectl describe -n kube-system po weave-net-abcde | grep "RANGE"`

search string is NOT the same across all CNIs. Look at the description.

### Service IP address
The best way to find service IP range is:
`cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep ip-range`
 Example output:  `- --service-cluster-ip-range=10.96.0.0/12`

### Where are the manifests and config files?
Exam cluster is based on kubeadm but uses `cri-o` instead of `docker`.
#### Manifests
`/etc/kubernetes/manifests`

#### Kubelet and Kubeproxy
Kubelet is a Linux service. To troubleshoot kubelet you need to ssh into the node. Location of `kubelet` config file on the node (in most cases): `/var/lib/kubelet`.

Kubeproxy (in most cases) is a Daemonset (one per node)

NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-proxy   3         3         3       3            3           kubernetes.io/os=linux   2d8h

configuration for kube-proxy is a ConfigMap (not a file on the disk). It is most difficult to troubleshoot is kube-proxy configuration.

#### PKI
`/etc/kubernetes/pki`
IMPORTANT NOTE: Scheduler, controller manager and kubelet ONLY has client certificate configurations (clients of kube-apiserver). Don't look for keys for these; you will only find .confs that resemble kubec config file. These are in `/etc/kubernetes` (not in `pki` directory)

`ca.key` - root CA private key for everything except ETCD
`ca.crt` - root CA certificate
`apiserver.key` - API server private key
`apiserver.crt` - API server certificate
`apiserver-etcd-client.crt` - client certificate for TLS communication between API Server (client) and ETCD
`apiserver-kubelet-client.crt` - client certificate for TLS communication between API server (client) and kubelet

## Pods
Watch out: for specific namespace in the question. Everything will get deployed onto `default` if you don't.

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
`k label nodes <node-name> <label-key>=<label-value>`
E.g.:
`k label nodes master type=master` if the question is asking for deployment on master node
`k label nodes master type=master` - if pod is to be scheduled on worker node

Step 2: check for taints on the node (especially master node)
`k describe nodes master | grep Taint`

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

### Resource requests for containers
```
spec:
  containers:
  - image: nginx
    name: pod1
    resources:
      requests: 
        cpu: 10m # 10 millicore CPUs
        memory: 10Mi # 10 Mebibytes (no it is not a spelling mistake)
```

### Mount volumes on pod
(namespace and DNS important)


### Set environment variables within pod
If it is just an environment variable
```
spec:
  containers:
  - image: nginx
    name: pod1
    resources: {}
    env: # add
    - name: MY_CERT
      value: cka
```
If the environment variable in based on existing kube object
```
spec:
  containers:
  - image: nginx
    name: pod1
    resources: {}
    env: # add
    - name: MY_NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
```
