## Kubernetes 101

### Getting to know your cluster:

### Overview of kubectl

Kubectl is a command line interface for running commands against Kubernetes clusters. `kubectl` looks for a file named config in the $HOME/.kube directory. You can specify other `kubeconfig` files by setting the KUBECONFIG environment variable or by setting the `--kubeconfig` flag.

### Syntax
Use the following syntax to run kubectl commands from your terminal window:

`kubectl [command] [TYPE] [NAME] [flags]`

where `command`, `TYPE`, `NAME`, and `flags` are:

* `command`: Specifies the operation that you want to perform on one or more resources, for example create, get, describe, delete.

* `TYPE`: Specifies the resource type. Resource types are case-insensitive and you can specify the singular, plural, or abbreviated forms. For example, the following commands produce the same output:
```
  kubectl get pod pod1
  kubectl get pods pod1
  kubectl get po pod1
```

* `NAME`: Specifies the name of the resource. Names are case-sensitive. If the name is omitted, details for all resources are displayed, for example `kubectl get pods`.

[Kubernetes Cheat Sheet](Kubernetes-Cheat-Sheet.pdf) - Courtesy of Linux Academy
