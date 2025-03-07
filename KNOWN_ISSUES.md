# Known issues

## Table of Contents

<!-- toc -->
- [Kubelet doesn't report metrics for all or subset of nodes](#kubelet-doesnt-report-metrics-for-all-or-subset-of-nodes)
- [Kubelet doesn't report pod metrics](#kubelet-doesnt-report-pod-metrics)
- [Incorrectly configured front-proxy certificate](#incorrectly-configured-front-proxy-certificate)
- [Network problem when connecting with Kubelet](#network-problem-when-connecting-with-kubelet)
<!-- /toc -->

## Kubelet doesn't report metrics for all or subset of nodes

**Symptoms**

If you run `kubectl top nodes` and not get metrics for all nods in the clusters or some nodes will return `<unknown>`  value. For example:
```
$ kubectl top nodes
NAME      CPU(cores)  CPU%       MEMORY(bytes)  MEMORY%
master-1  192m        2%         10874Mi        68%
node-1    582m        7%         9792Mi         61%
node-2    <unknown>   <unknown>  <unknown>      <unknown>
```

**Debugging**

Please check if your Kubelet returns correct node metrics.

- Version 0.5.x and earlier

You can do that by checking Summary API on nodes that are missing metrics.

You can read JSON response returned by command below and check `.node.cpu` and `.node.memory` fields. 
```console
NODE_NAME=<Name of node in your cluster>
kubectl get --raw /api/v1/nodes/$NODE_NAME/proxy/stats/summary
```

Alternativly you can run one liner using (requires [jq](https://stedolan.github.io/jq/)):
```
NODE_NAME=<Name of node in your cluster>
kubectl get --raw /api/v1/nodes/$NODE_NAME/proxy/stats/summary | jq '{cpu: .node.cpu, memory: .node.memory}'
```

If usage values are equal zero, means that there is problem is related to Kubelet and not Metrics Server. Metrics Server requires that `.cpu.usageCoreNanoSeconds` or `.memory.workingSetBytes` to not be zero. Example of invalid result:
```json
{
  "cpu": {
    "time": "2022-01-19T13:12:56Z",
    "usageNanoCores": 0,
    "usageCoreNanoSeconds": 0
  },
  "memory": {
    "time": "2022-01-19T13:12:56Z",
    "availableBytes": 16769261568,
    "usageBytes": 0,
    "workingSetBytes": 0,
    "rssBytes": 0,
    "pageFaults": 0,
    "majorPageFaults": 0
  }
}
```

- Version 0.6.x and later

You can do that by checking resource metrics on nodes that are missing metrics.

```console
NODE_NAME=<Name of node in your cluster>
kubectl get --raw /api/v1/nodes/$NODE_NAME/proxy/metrics/resource
```

If usage values are equal zero or timestamp is lost, means that there is problem is related to Kubelet and not Metrics Server. Metrics Server require value and timestamp of  `node_cpu_usage_seconds_total`  `node_memory_working_set_bytes`  `container_cpu_usage_seconds_total` and `container_memory_working_set_bytes` to not be zero or missed. 

**Known causes**

* Nodes use cgroupv2 that are not supported as of Kubernetes 1.21.

**Workaround**

* Reconfigure/rollback the nodes to use use cgroupv1. For non-production clusters you might want to alternatily try out cgroupv2 alpha support in Kubernetes v1.22 https://github.com/kubernetes/enhancements/issues/2254.

## Kubelet doesn't report pod metrics

If you run `kubectl top pods` and not get metrics for the pod, even though pod is already running for over 1 minute. To confirm please check that Metrics Server has logged that there were not metrics available for pod.

You can get Metrics Server logs by running `kubectl -n kube-system logs -l k8s-app=metrics-server` (works with official manifests, if you installed Metrics Server different way you might need to tweak the command). Example log line you are looking for:
```
W1106 20:50:15.238493 73592 top_pod.go:265] Metrics not available for pod default/example, age: 22h29m11.238478174s
error: Metrics not available for pod default/example, age: 22h29m11.238478174s
```

**Debugging**

Please check if your Kubelet is correctly returning pod metrics. 

- Version 0.5.x and earlier

You can do that by checking Summary API on node where pod with missing metrics is running (can be checked by running `kubectl -n <pod_namespace> describe pod <pod_name>`:
```console
NODE_NAME=<Name of node where pod runs>
kubectl get --raw /api/v1/nodes/$NODE_NAME/proxy/stats/summary
```

This will return JSON that will have two keys `node` and `pods`. 
Empty list of pods means that problem is related to Kubelet and not to Metrics Server.

One liner for number of pod metrics reported by first node in cluster (requires [jq](https://stedolan.github.io/jq/)):
```console
kubectl get --raw /api/v1/nodes/$(kubectl get nodes -o json  | jq -r '.items[0].metadata.name')/proxy/stats/summary | jq '.pods | length'
```

- Version 0.6.x and later

You can do that by checking resource metrics on node where pod with missing metrics is running (can be checked by running `kubectl -n <pod_namespace> describe pod <pod_name>`:
```console
NODE_NAME=<Name of node in your cluster>
kubectl get --raw /api/v1/nodes/$NODE_NAME/proxy/metrics/resource
```

**Known causes**

* [Kubelet doesn't report pod metrics in Kubernetes 1.19 with Docker <v19](https://github.com/kubernetes/kubernetes/issues/94281)
  
  **Workaround**
  
  Upgrade Docker to v19.03
  

## Incorrectly configured front-proxy certificate

**Symptoms**


**Debuging**

Please check if your metrics server reports problem with authenticating client certificate, in particular errors mentioning `Unable to authenticate the request due to an error`. You can check logs by running command:
```
kubectl logs -n kube-system -l k8s-app=metrics-server --container metrics-server
```

Problem with front-proxy certificate can be recognized if logs have line similar to one below:
```
E0524 01:37:36.055326       1 authentication.go:65] Unable to authenticate the request due to an error: x509: certificate signed by unknown authority
```

**Known Causes**

* kubeadm uses separate `front-proxy` certificates that are not signed by main cluster certificate authority. 

  To fix this problem you need to provide kube-apiserver proxy-client CA to Metrics Server under `--requestheader-client-ca-file` flag. You can read more about this flag in [Authenticating Proxy](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#authenticating-proxy)




  1. Find your front-proxy certificates by checking arguments passed in kube-apiserver config (by default located in /etc/kubernetes/manifests/kube-apiserver.yaml)
  
    ```
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    ```

  2. Create configmap including `front-proxy-ca.crt`

    ```
    kubectl -nkube-system create configmap front-proxy-ca --from-file=front-proxy-ca.crt=/etc/kubernetes/pki/front-proxy-ca.crt -o yaml | kubectl -nkube-system replace configmap front-proxy-ca -f -
    ```

  3. Mount configmap in Metrics Server deployment and add `--requestheader-client-ca-file` flag

  ```
        - args:
          - --requestheader-client-ca-file=/ca/front-proxy-ca.crt // ADD THIS!
          - --cert-dir=/tmp
          - --secure-port=4443
          volumeMounts:
          - mountPath: /tmp
            name: tmp-dir
          - mountPath: /ca // ADD THIS!
            name: ca-dir

        volumes:
        - emptyDir: {}
          name: tmp-dir
        - configMap: // ADD THIS!
            defaultMode: 420
            name: front-proxy-ca
          name: ca-dir
  ```

## Network problem when connecting with Kubelet

**Symptoms**

When running `kubectl top nodes` we get partial or no information. For example results like:
```
NAME         CPU(cores) CPU%      MEMORY(bytes)   MEMORY%     
k8s-node01   59m        5%        1023Mi          26%         
k8s-master   <unknown>  <unknown> <unknown>       <unknown>               
k8s-node02   <unknown>  <unknown> <unknown>       <unknown>         
```

**Debugging**

Please check if your metrics server reports problems with connecting to Kubelet address, in particular errors will include `dial tcp IP(or hostname):10250: i/o timeout`. You can check logs by running command:

```
kubectl logs -n kube-system -l k8s-app=metrics-server --container metrics-server
```

Problem with network can be recognized if logs have line similar to one below:
```
unable to fully collect metrics: [unable to fully scrape metrics from source kubelet_summary:k8s-master: unable to fetch metrics from Kubelet k8s-master
(192.168.17.150): Get https://192.168.17.150:10250/stats/summary?only_cpu_and_memory=true: dial tcp 192.168.17.150:10250: i/o timeout
```

**Known solutions**
* [Calico] Check whether the value of `CALICO_IPV4POOL_CIDR` in the calico.yaml conflicts with the local physical network segment. The default: `192.168.0.0/16`.

  See [Kubernetes in Calico] for more details.

[Kubernetes in Calico]: https://docs.projectcalico.org/getting-started/kubernetes/quickstart

* [Firewall rules/Security groups] Check firewall configuration on your nodes. The reason may be in closed ports for incoming connections, so the metrics server cannot collect metrics from other nodes.
