The whole point of BIG-IP Next for Kubernetes is the ability to control application delivery to Kubernetes services. Let's create a service in our red tenant.
## Todo: Create a red tenant Deployment and Service

We will deploy a nginx TCP demonstration pod as a Kubernetes `Deployment` and then create a simple `Service` pointing to our `Deployment` give it a stable IP address. 

#### Run: `kubectl apply -f resources/nginx-red-deployment.yaml`

```
kubectl apply -f resources/nginx-red-deployment.yaml
```

```
deployment.apps/nginx-deployment created
service/nginx-app-svc created
```

## Todo: Create ingress GatewayType, Gateway, and TCPRoute

Now the fun part. Put on your NetOps hat and create a Gateway API `GatewayClass` and `Gateway` resources for our service. The NetOps user gets to decided what BIG-IP Next for Kubernetes instance to associate `Gateway`s by creating a `GatewayClass`.

Next the NetOps user get to decide data center addresses and listener port to expose our `Service` to the outside world on red's VLAN. 
### Show: resources/nginx-red-gw-api.yaml NetOps Part 

```
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: f5-gateway-class
  namespace: red
spec:
  controllerName: "f5.com/f5-gateway-controller"
  description: "F5 BIG-IP Kubernetes Gateway"
```

```
apiVersion: gateway.k8s.f5net.com/v1
kind: Gateway
metadata:
  name: my-l4route-tcp-gateway
  namespace: red
spec:
  addresses:
  - type: "IPAddress"
    value: 198.19.19.100
  gatewayClassName: f5-gateway-class
  listeners:
  - name: nginx
    protocol: TCP
    port: 80
    allowedRoutes:
      kinds:
      - kind: L4Route
```

Then as a DevOps person, we create a Gateway `L4Route` and use its `parentRefs` attribute to say what `Gateway` it should use. 
### Show: resources/resources/nginx-red-gw-api.yaml DevOps Part 

```
apiVersion: gateway.k8s.f5net.com/v1
kind: L4Route
metadata:
  name: l4-tcp-app
  namespace: red
spec:
  protocol: TCP
  parentRefs:
  - name: my-l4route-tcp-gateway
    sectionName: nginx
  rules:
  - backendRefs:
    - name: nginx-app-svc
      namespace: red
      port: 80
```

Let's deploy these resources in Kubernetes.
#### Run: `kubectl apply -f `resources/nginx-red-gw-api.yaml

```
kubectl apply -f resources/nginx-red-gw-api.yaml
```

```
gatewayclass.gateway.networking.k8s.io/f5-gateway-class created
gateway.gateway.k8s.f5net.com/my-l4route-tcp-gateway created
l4route.gateway.k8s.f5net.com/l4-tcp-app created
```

## Todo: Test BIG-IP Next for Kubernetes ingress

We are going to run a `curl` web client command from our docker deployed infra-client-1 container and see if we can hit the virtual server we created in BIG-IP for 198.19.19.100 in our `Gateway` resource in the last step.

![[Testing Ingress for red.png]]

#### Run: `docker exec -ti infra-client-1 curl -I http://198.19.19.100

```
docker exec -ti infra-client-1 curl -I http://198.19.19.100
```

```
HTTP/1.1 200 OK
Server: nginx/1.27.4
Date: Thu, 20 Feb 2025 18:04:34 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Wed, 05 Feb 2025 11:06:32 GMT
Connection: keep-alive
ETag: "67a34638-267"
Accept-Ranges: bytes
```

How did it get there? Let's see what the router container infra-frr-1 between the infra-client-1 and the BIG-IP Next instances has been peered with.

#### Run: `docker exec -ti infra-frr-1 vtysh -c "show bgp summary"`

```
docker exec -ti infra-frr-1 vtysh -c "show bgp summary"
```

```
IPv4 Unicast Summary (VRF default):
BGP router identifier 192.0.2.250, local AS number 65500 vrf-id 0
BGP table version 7
RIB entries 11, using 2112 bytes of memory
Peers 3, using 2151 KiB of memory
Peer groups 1, using 64 bytes of memory

Neighbor           V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
*192.0.2.201       4      64443       376       379        0    0    0 03:06:11            3        6 N/A
*192.0.2.202       4      64443       376       379        0    0    0 03:06:18            3        6 N/A
*2001::192:0:2:202 4      64443        13        14        0    0    0 00:05:06        NoNeg    NoNeg N/A

Total number of neighbors 3
* - dynamic neighbor
3 dynamic neighbor(s), limit 100

IPv6 Unicast Summary (VRF default):
BGP router identifier 192.0.2.250, local AS number 65500 vrf-id 0
BGP table version 2
RIB entries 3, using 576 bytes of memory
Peers 3, using 2151 KiB of memory
Peer groups 1, using 64 bytes of memory

Neighbor           V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
*192.0.2.201       4      64443       376       379        0    0    0 03:06:11        NoNeg    NoNeg N/A
*192.0.2.202       4      64443       376       379        0    0    0 03:06:18        NoNeg    NoNeg N/A
*2001::192:0:2:202 4      64443        13        14        0    0    0 00:05:06            2        2 N/A

Total number of neighbors 3
* - dynamic neighbor
3 dynamic neighbor(s), limit 100
```

Notice both BIG-IP Next instances, 192.168.2.201 and 192.168.2.202 are peered to our router!

What did our BIG-IP Next instances advertise for our red service virtual service?
#### Run: `docker exec -ti infra-frr-1 vtysh -c "show ip route"`

```
docker exec -ti infra-frr-1 vtysh -c "show ip route"
```

```
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

K>* 0.0.0.0/0 [0/0] via 198.51.100.1, eth0, 03:26:14
C>* 192.0.2.0/24 is directly connected, eth1, 03:26:14
B>* 192.0.2.100/32 [20/0] via 192.0.2.201, eth1, weight 1, 03:08:51
B>* 192.0.2.101/32 [20/0] via 192.0.2.202, eth1, weight 1, 03:09:04
B>* 192.0.2.110/32 [20/0] via 192.0.2.201, eth1, weight 1, 03:08:51
B>* 192.0.2.111/32 [20/0] via 192.0.2.202, eth1, weight 1, 03:09:04
B>* 198.19.19.100/32 [20/0] via 192.0.2.201, eth1, weight 1, 00:14:18
  *                         via 192.0.2.202, eth1, weight 1, 00:14:18
C>* 198.51.100.0/24 is directly connected, eth0, 03:26:14
```

### Class Discuss: ECMP base ingress routing with BIG-IP Next to pod IP Endpoints with routing to node IPs

Our virtual server address, set by our NetOps user, can be reached at all BIG-IP Next instances peered to the router (we only have two).  What happens then?

The BIG-IP Next instances could build pool members from `ClusterIP` addresses representing our service, forward to one of them and let the `kube-proxy` instance on a node proxy through to a `Endpoint` pod IP as it does for requests made inside the cluster.  

#### Run: `kubectl get service -n red`

```
kubectl get service -n red
```

```
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx-app-svc   ClusterIP   10.96.157.55   <none>        80/TCP    4m
```

However, if we did that, it would mean our solution would be wasting CPU resources watching `kube-proxy` use linux kernel `netfilter` table NAT rules (`iptables`) to get you to the pod hosting an instance of your application `Endpoint`. 

BIG-IP Next for Kubernetes instead discovers the `Endpoint` pod IPs associated with the `Service`, builds a pool of pod IP address, discovers which nodes a given pod is deployed, and then routes the load balanced request to the right node IP withe the destination address of the pod IP pool member.

#### Run: `kubectl get endpoints -n red`

```
kubectl get endpoints -n red
```

```
NAME            ENDPOINTS           AGE
nginx-app-svc   10.244.227.201:80   5m
```

This removes the `kube-proxy` overhead for our ingress traffic. We keep telling everyone that we are saving significant CPU cycles. Now you know why!

## Todo: Check Egress from red tenant container

Let's see what SNAT IP we put on traffic coming from our red tenant.

#### Run: `kubectl describe f5-spk-snatpool red-snat`

```
kubectl describe f5-spk-snatpool red-snat
```

```
Name:         red-snat
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  k8s.f5net.com/v1
Kind:         F5SPKSnatpool
Metadata:
  Creation Timestamp:  2025-02-20T15:05:18Z
  Finalizers:
    handletmmconfig_inconsistency
  Generation:        1
  Resource Version:  6173
  UID:               923fe787-13bc-44c0-bf19-678ca38ab198
Spec:
  Address List:
    [192.0.2.100 2001::192:0:2:100]
    [192.0.2.101 2001::192:0:2:101]
  Name:                         red-snat
  Shared Snat Address Enabled:  false
Status:
  Conditions:
    Last Transition Time:  2025-02-20T15:05:18Z
    Message:
    Observed Generation:   0
    Reason:                Accepted
    Status:                True
    Type:                  Accepted
    Last Transition Time:  2025-02-20T15:05:18Z
    Message:               CR config sent to all grpc endpoints
    Observed Generation:   2
    Reason:                Programmed
    Status:                True
    Type:                  Programmed
  Generation Id:           0
Events:                    <none>
```

If we did our job right we can generate traffic from the pod in the red namespace and it should show up at the infra-client-1 container from either 192.0.2.100 or 192.0.2.101. 

![[Egress Tenancy.png]]

We will generate a outbound web request from the red pod and let the `infra-client-1` web service tell us what it sees about the request. We'll run a `curl` web client request through Kubernetes.
#### Run: `kubectl exec -ti -n red deploy/nginx-deployment -- curl http://198.51.100.100/txt`

```
kubectl exec -ti -n red deploy/nginx-deployment -- curl http://198.51.100.100/txt
```

```
================================================
 ___ ___   ___                    _
| __| __| |   \ ___ _ __  ___    /_\  _ __ _ __
| _||__ \ | |) / -_) '  \/ _ \  / _ \| '_ \ '_ \
|_| |___/ |___/\___|_|_|_\___/ /_/ \_\ .__/ .__/
                                      |_|  |_|
================================================

      Node Name: F5 Docker vLab
     Short Name: nginx

      Server IP: 198.51.100.100
    Server Port: 80

      Client IP: 192.0.2.100
    Client Port: 62899

Client Protocol: HTTP
 Request Method: GET
    Request URI: /txt

    host_header: 198.51.100.100
     user-agent: curl/7.88.1
```

Yeah! We have egress requests from pods in our 'red' tenant namespace are having their traffic SNAT applied appropriately! 

If we wanted to see the address shift to 192.0.2.101, we can continue to make request, but we will have to wait for the ECMP packet flow to send us through a new forwarding virtual server on the BIG-IP. You can repeat the above command until you see it shift.

Let's try our blue tenant. We have the complete deployment for the blue tenant, pod `Deployment`, `Service`, `Gateway`, `TCPRoute` in one YAML file we can deploy with one command.

#### Run: `kubectl apply -f ./resources/nginx-blue-deployment.yaml`

```
kubectl apply -f ./resources/nginx-blue-deployment.yaml
```

```
deployment.apps/nginx-deployment created
service/nginx-app-svc created
gateway.gateway.k8s.f5net.com/my-l4route-tcp-gateway created
l4route.gateway.k8s.f5net.com/l4-tcp-app created
```

Let's see what the SNAT pool for blue looks like.
#### Run: `kubectl describe f5-spk-snatpool red-snat`

```
kubectl describe f5-spk-snatpool blue-snat
```

```
Name:         blue-snat
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  k8s.f5net.com/v1
Kind:         F5SPKSnatpool
Metadata:
  Creation Timestamp:  2025-02-20T23:53:05Z
  Finalizers:
    handletmmconfig_inconsistency
  Generation:        1
  Resource Version:  4936
  UID:               20ddfd35-adcc-4d38-8efb-4a7beaeef442
Spec:
  Address List:
    [192.0.2.110 2001::192:0:2:110]
    [192.0.2.111 2001::192:0:2:111]
  Name:                         blue-snat
  Shared Snat Address Enabled:  false
Status:
  Conditions:
    Last Transition Time:  2025-02-20T23:53:05Z
    Message:
    Observed Generation:   0
    Reason:                Accepted
    Status:                True
    Type:                  Accepted
    Last Transition Time:  2025-02-20T23:53:06Z
    Message:               CR config sent to all grpc endpoints
    Observed Generation:   2
    Reason:                Programmed
    Status:                True
    Type:                  Programmed
  Generation Id:           0
Events:
  Type    Reason         Age   From            Message
  ----    ------         ----  ----            -------
  Normal  Added/Updated  20m   spk-controller  F5Snatpool default/blue-snat was added/updated
```

So we should see the blue tenant make requests from 192.168.2.110 or 192.168.2.111. Let's test.

#### Run: `kubectl exec -ti -n red deploy/nginx-deployment -- curl http://198.51.100.100/txt`

```
kubectl exec -ti -n blue deploy/nginx-deployment -- curl http://198.51.100.100/txt
```

```
================================================
 ___ ___   ___                    _
| __| __| |   \ ___ _ __  ___    /_\  _ __ _ __
| _||__ \ | |) / -_) '  \/ _ \  / _ \| '_ \ '_ \
|_| |___/ |___/\___|_|_|_\___/ /_/ \_\ .__/ .__/
                                      |_|  |_|
================================================

      Node Name: F5 Docker vLab
     Short Name: nginx

      Server IP: 198.51.100.100
    Server Port: 80

      Client IP: 192.0.2.111
    Client Port: 10764

Client Protocol: HTTP
 Request Method: GET
    Request URI: /txt

    host_header: 198.51.100.100
     user-agent: curl/7.88.1
```

Logging, observability, and firewall rules can now identify our Kubernetes 'red' and 'blue' tenants by simply checking the source IPs of the egress traffic coming from their workloads.

Think how important this type of traffic network segmentation is when we are trying to secure traffic from tenant sharing expensive GPUs in a cluster, but making request for objects as part of a AI RAG (retrieval augmented generation) pulling in data from a particular corpus of policy documents. You need the network segmentation to guarantee security.    
## Todo: Explore BIG-IP Next telemetry through Kabana

We will access the deployed Grafana web user interface through the link provided as part of the lab. Open a browser to your lab Grafana URL.

![[Grafana Login.png]]

The default credentials are:

username: `admin`
password: `admin`

You will be prompted to change the password. Go ahead and do that or Skip it. 

Navigate to Dashboard and then load the F5 BNK Dashboard.  You will see there are some example Visualization defined for TMM (data path), ACLs, and then per 'Red' and 'Blue' tenants. 

You've already seen the commands to generate traffic which ingresses and egress Red and Blue tenants. 

See if you can use `docker exec` to generate traffic from the `infra-client-1` ingressing the cluster to your red (http://198.19.19.100/txt) or blue (http://198.20.20.100/txt) virtual servers. Observe  their traffic visualizations. 

See if you can use `kubectl exec` to generate traffic from the 'red' and 'blue' pods to your egressing towards your `infra-client-1` (http:/http://198.51.100.100/txt) web services. Observe their traffic visualizations. 

In our lab we demonstrated a few things:

1) Kubernetes cluster concepts and details which make BIG-IP Next for Kubernetes valuable
2) How BIG-IP Next for Kubernetes is installed and works with the infrastructure and services
3) How Kubernetes workloads deployed in Kubernetes namespace tenants get BIG-IP Next for Kubernetes application delivery and security for both ingress and egress traffic.
