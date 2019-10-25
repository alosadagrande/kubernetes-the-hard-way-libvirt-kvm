# Smoke Test

In this lab you will complete a series of tasks to ensure your Kubernetes cluster is functioning correctly.

## Data Encryption

In this section you will verify the ability to [encrypt secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted).

Create a generic secret:

```
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Print a hexdump of the `kubernetes-the-hard-way` secret stored in etcd:

```
kcli ssh master00 \
  "sudo ETCDCTL_API=3 /usr/local/bin/etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"
```

> output

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a bf 0d 36 4f ed 16 84  |:v1:key1:..6O...|
00000050  01 00 ea 28 51 5d 28 c0  d2 c5 44 47 87 3e 8f cc  |...(Q](...DG.>..|
00000060  1b f2 f8 56 f2 8c 89 e8  4f fc 87 aa d6 ee 6e 25  |...V....O.....n%|
00000070  71 d0 aa e2 a3 70 84 78  af 4c d6 71 d9 7b 59 93  |q....p.x.L.q.{Y.|
00000080  24 37 93 33 bd 19 2e 82  0b 68 68 d8 9f 7e 1f 4b  |$7.3.....hh..~.K|
00000090  7d cf ad f2 e9 2f 8f 7b  ba 53 4f 2b 35 08 34 63  |}..../.{.SO+5.4c|
000000a0  a9 54 d3 a0 6a 77 ed 32  38 b9 ba 62 0a 33 39 72  |.T..jw.28..b.39r|
000000b0  23 6d 8f 9e 44 39 7d 70  4f 98 ca 8f 30 d8 5c 70  |#m..D9}pO...0.\p|
000000c0  f4 19 31 5f 66 f2 f2 13  86 63 1d 61 ad 26 a5 eb  |..1_f....c.a.&..|
000000d0  a7 5b dc 09 80 99 69 9d  78 71 a8 ff cc 72 6a 02  |.[....i.xq...rj.|
000000e0  2d c7 ad f5 07 e1 fe 6e  55 0a                    |-......nU.|
000000ea

```

The etcd key should be prefixed with `k8s:enc:aescbc:v1:key1`, which indicates the `aescbc` provider was used to encrypt the data with the `key1` encryption key.

## Deployments

In this section you will verify the ability to create and manage [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Create a deployment for the [nginx](https://nginx.org/en/) web server:

```
kubectl create deployment nginx --image=nginx
```

List the pod created by the `nginx` deployment:

```
kubectl get pods -l app=nginx -o wide
```

> output

```
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE                     NOMINATED NODE   READINESS GATES
nginx-86c57db685-h82f2   1/1     Running   0          90s   10.200.2.17   worker02.k8s-thw.local   <none>           <none>
```

### Port Forwarding

In this section you will verify the ability to access applications remotely using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Retrieve the full name of the `nginx` pod:

```
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
```

Forward port `8080` on your local machine to port `80` of the `nginx` pod:

```
kubectl port-forward $POD_NAME 8080:80
```

> output

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

In a new terminal make an HTTP request using the forwarding address:

```
curl -IL http://localhost:8080 
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.17.5
Date: Fri, 25 Oct 2019 07:59:07 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 22 Oct 2019 14:30:00 GMT
Connection: keep-alive
ETag: "5daf1268-264"
Accept-Ranges: bytes
```

Switch back to the previous terminal and stop the port forwarding to the `nginx` pod:

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### Logs

In this section you will verify the ability to [retrieve container logs](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

Print the `nginx` pod logs:

```
kubectl logs $POD_NAME
```

> output

```
127.0.0.1 - - [25/Oct/2019:07:58:11 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
127.0.0.1 - - [25/Oct/2019:07:58:18 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
127.0.0.1 - - [25/Oct/2019:07:58:19 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
127.0.0.1 - - [25/Oct/2019:07:58:43 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
127.0.0.1 - - [25/Oct/2019:07:59:07 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.29.0" "-"
```

### Exec

In this section you will verify the ability to [execute commands in a container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container).

Print the nginx version by executing the `nginx -v` command in the `nginx` container:

```
kubectl exec -ti $POD_NAME -- nginx -v
```

> output

```
nginx version: nginx/1.17.5
```

## Services

In this section you will verify the ability to expose applications using a [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Expose the `nginx` deployment using a [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) service:

```
kubectl expose deployment nginx --port 80 --type NodePort
```
Verify the creation of a service object and a NodePort mapping

```
kubectl get svc -lapp=nginx
NAME    TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.32.0.24   <none>        80:31459/TCP   53s
```

Retrieve the node port assigned to the `nginx` service:

```
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Verify in your deployment that $NODE_PORT is allowed in your firewall rules. In this case it is the 31459/tcp, but in your deployment probably would be any other in a similar range.


Retrieve the external IP address of any worker instance. Note that it can be any of the worker and not necessarily the worker where the nginx application is running. Actually the pod is running on worker02 and we are retrieving the IP address of worker00.

```
EXTERNAL_IP=$(kcli info vm worker00 | grep "ip:" | awk {'print$2'})
```

Make an HTTP request using the external IP address and the `nginx` node port:

```
curl -I http://${EXTERNAL_IP}:${NODE_PORT}
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.17.5
Date: Fri, 25 Oct 2019 08:09:07 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 22 Oct 2019 14:30:00 GMT
Connection: keep-alive
ETag: "5daf1268-264"
Accept-Ranges: bytes

```

Next: [Cleaning Up](14-cleanup.md)
