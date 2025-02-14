Here's a quick list of commands:

    Nodes:
    kubectl get nodes
    (Shows node status and roles)

    Pods (all namespaces):
    kubectl get pods -A
    (Lists all pods and their statuses)

    Services (all namespaces):
    kubectl get services -A
    (Displays service info)

    Events:
    kubectl get events -A
    (Helps spot issues with warnings and errors)

    Cluster Info:
    kubectl cluster-info
    (Provides basic cluster endpoint info)

You can also use:

    Detailed node info:
    kubectl describe node <node-name>
    Pod logs:
    kubectl logs <pod-name> -n <namespace>

These should help you check the cluster health easily.