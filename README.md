# CKA Shell Setup

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
Command shortcuts

0. Linux tricks

Erase beginning of a string in a line to retain just the name
E.g.: in `current-context: context01`, to only extract the name of the context,

`cat ~/.kube/config | grep current-context | sed -e "s/current-context: //"

1. View kubeconfig as yaml

Using kubectl
```
k config view -o yaml
k config view -o jsonpath="{.contexts[*].name}"
k config view -o jsonpath="{.contexts[*].name}" | tr " " "\n" # new lines
```
Get current context without using kubectl (parse .kube/config)

```
cat ~/.kube/config | grep current
```
