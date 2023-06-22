# Block Source IP using RBAC

Demonstrates how to use an `EnvoyFilter` to deny requests from an IP in the network filter.
This uses the native RBAC filter.

## Prerequisites

1. Create the required env vars

    ```
    export CLUSTER_OWNER="kasunt"
    export PROJECT="source_ip_deny"
    ```

2. Provision a local cluster (Using custom built `colima`)

    ```
    colima start --runtime containerd  -p $PROJECT -c 4 -m 8 -d 20 --network-address --install-metallb --metallb-address-pool "192.168.106.230/29" --kubernetes --kubernetes-disable traefik,servicelb --kubernetes-version v1.24.13+k3s1
    ```

3. Install `istioctl` version `1.17.3`.

## Setting Up Istio

```
istioctl operator init
kubectl apply -f istio-provision.yaml
```

## Deploy App and Config

```
# Deploy sample application
kubectl apply -f apps/httpbin/ns.yaml
kubectl create ns istio-config
kubectl -n httpbin apply -f apps/httpbin/deploy.yaml

# Various configuration
kubectl apply -f config/httpbin-igw-vs.yaml
kubectl apply -f config/source-ip-deny-envoyfilter.yaml
```

## Testing

1. Run an external `curl`.

    ```
    curl -iv -H "Host: httpbin.test.com" http://192.168.106.224/get
    ```

    Should result in for instance,
    ```
    *   Trying 192.168.106.224:80...
    * Connected to 192.168.106.224 (192.168.106.224) port 80 (#0)
    > GET /get HTTP/1.1
    > Host: httpbin.test.com
    > User-Agent: curl/7.88.1
    > Accept: */*
    >
    * Empty reply from server
    * Closing connection 0
    curl: (52) Empty reply from server
    ```

    Packet capture output of the request being denied,
    ![Screenshot 2023-06-22 at 14 20 52](https://github.com/pseudonator/useful-istio-envoyfilter-samples/assets/2648624/f4e5499e-3b0e-4e6e-b345-22747a560539)

    Comparison with a working request,
    ![Screenshot 2023-06-22 at 14 22 04](https://github.com/pseudonator/useful-istio-envoyfilter-samples/assets/2648624/39afa160-9420-4410-a4ac-0212e251c978)


3. Check if the stat counters have also changed for this particular RBAC policy.

    ```
    kubectl exec deploy/istio-ingressgateway -n istio-system -- curl -s http://localhost:15000/stats | grep "source_ip_blocker"
    ```

    Should result in `denied` counter increasing whenever the policy is triggered,
    ```
    source_ip_blockerrbac.allowed: 1
    source_ip_blockerrbac.denied: 14
    source_ip_blockerrbac.shadow_allowed: 0
    source_ip_blockerrbac.shadow_denied: 0
    ```
