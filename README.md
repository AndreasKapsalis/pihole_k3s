This guide show how to install the [PiHole](https://pi-hole.net/) DNS sinkhole in a K3S cluster. 
We will be using a K3S cluster using MetalLB and the Nginx ingress controller instead of the default ServiceLB and Traefik options. There is no particular reason behind this other than I have better experience with the Nginx ingress controller, and that MetalLB is gaining traction. 
Having said that this setup should work with ServiceLB and Traefik with minimal modifications. 
This [guide](https://kauri.io/38-install-and-configure-a-kubernetes-cluster-with/418b3bc1e0544fbc955a4bbba6fff8a9/a) shows how to install K3S with MetalLB and the Nginx ingress controller.

# 1. Provision persistent storage
We have the option to allocate persistent storage for the PiHole and dnsmaqd services configuration files. We can start by creating two `PeristentVolume` resources of type `hostPath`:

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pihole-etc-pv
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /tmp/pihole-etc
    type: DirectoryOrCreate
```
and
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pihole-dnsmasq-pv
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /tmp/pihole-dnsmasq
    type: DirectoryOrCreate
```
Create the resources with:

```
kubectl create -f pihole-etc-pv.yaml
kubectl create -f pihole-dnsmasq-pv.yaml
```
Next create the appropriate `PersistentVolumeClaim` resources:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pihole-etc-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeName: pihole-etc-pv
  storageClassName: ""
```
and 
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pihole-dnsmasq-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeName: pihole-dnsmasq-pv
  storageClassName: ""
```
Note here that we specifically set the storage class name to `""` and we specify the name of the persistent volume the claim will be bound to, to avoid using the local autoprovisioner class. 

Create the resources with:

```
kubectl create -f pihole-etc-pvc.yaml
kubectl create -f pihole-dnsmasq-pvc.yaml
```
Validate that the resources were created without any errors:
```
$ kubectl get pv
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS   REASON   AGE
pihole-etc-pv       1Gi        RWO            Retain           Bound    default/pihole-etc-claim                               4m
pihole-dnsmasq-pv   1Gi        RWO            Retain           Bound    default/pihole-dnsmasq-claim                           4m
```
and 
```
$ kubectl get pvc
NAME                   STATUS   VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pihole-etc-claim       Bound    pihole-etc-pv       1Gi        RWO                           3m
pihole-dnsmasq-claim   Bound    pihole-dnsmasq-pv   1Gi        RWO                           3m
```

# 2. Create the deployment
Now that we have provisioned the storage we can move to deploy the `PiHole` application.
But before we do that we will need to set a password for the PiHole admin dashboard. For that we can create a `Secret` resource:

```
kubectl create secret generic pihole-pass --from-literal=password=<YOURPASSWORD>
```

Validate that the resource was created successfully:
```
$ kubectl get secret pihole-pass
NAME          TYPE     DATA   AGE
pihole-pass   Opaque   1      38s
```

Now we can create the application using a `Deployment`:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: pihole
  name: pihole
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pihole
  template:
    metadata:
      labels:
        app: pihole
    spec:
      hostNetwork: true
      nodeSelector:
        kubernetes.io/hostname: pi-worker
      volumes:
      - name: pihole-local-etc-volume
        persistentVolumeClaim:
          claimName: pihole-etc-claim
      - name: pihole-local-dnsmasq-volume
        persistentVolumeClaim:
          claimName: pihole-dnsmasq-claim
      containers:
      - image: pihole/pihole:latest
        name: pihole
        env:
        - name: WEBPASSWORD
          valueFrom:
            secretKeyRef:
              name: pihole-pass
              key: password
        - name: TZ
          value: "Europe/London"
        volumeMounts:
        - name: pihole-local-etc-volume
          mountPath: "/etc/pihole"
        - name: pihole-local-dnsmasq-volume
          mountPath: "/etc/dnsmasq.d"
```
Notice the following:
- We set `hostNetwork` to `true`. We need the pihole service to listen to UDP port `53`, if we want to server clients outside the pod cluster network. That way the pod will be attached to the hosts' network namespace. It is crucial that we do not schedule a pod binding to UDP port `53` on the same host. 
- We specifically select the host node by using the appropriate `nodeSelector`. This way we avoid the pod being scheduled on another node, and thus receiving a different IP. This is far from optimal, but setting a HA deployment behind a Layer-2 LB is out of the scope of this guide

Now create the deployment with:
```
$ kubectl create -f pihole-deployment.yaml
```
Validate that the resources were created successfully:

```
$ kubectl get deployments.apps pihole 
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
pihole   1/1     1            1           2m
```
Check the pod of the deployment:
```
$ kubectl get pods -l app=pihole -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP              NODE        NOMINATED NODE   READINESS GATES
pihole-65f94c5fb-djh6m   1/1     Running   0          2m   192.168.1.103   pi-worker   <none>           <none> 
```
Notice that the pod has assumed the private IP of the worker node. **This is the IP that you would need to use when configuring your device primary DNS resolver**

# 3. Expose the admin dashboard
Since you created the pod using `hostNetwork`:`true` this mean that it will be created within the host's network namespace, binding to UDP port 53 and TCP port 80 of the host. You can then access the PiHole dashboard just by pointing your browser to the private IP of the host. 
However this limits the options for connecting to the dashboard, plus the connection would not be secure, plus it's not fun

Instead we can create an ingress that will use a self-signed certificate to point to the admin dashboard using the provisioned LB by MetalLB. Plus, with some clever NATting or if your network has access to a public IP address pool you can use that to connect to the dashboard outside your private network.

First you need to expose the deployment using a `Service` of type `ClusterIP`:
```
apiVersion: v1
kind: Service
metadata:
  name: pihole-svc
spec:
  selector:
    app: pihole
  ports:
  - name: pihole-admin
    port: 80
    targetPort: 80
  type: ClusterIP
```
Create the resource:
```
$ kubectl create -f pihole-svc.yaml
```
Validate that the resource is created and check that the endpoints have been populated:
```
$ kubectl describe svc pihole-svc
Name:              pihole-svc
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=pihole
Type:              ClusterIP
IP:                10.43.23.13
Port:              pihole-admin  80/TCP
TargetPort:        80/TCP
Endpoints:         192.168.1.103:80
Session Affinity:  None
Events:            <none>
```

Before we create the ingress resource we will need to create a self-signed certificate for the ingress. We will do that using OpenSSL.
- Generate the private key:
```
$ openssl genrsa -out pihole.key 2048
```
- Then create signing request:
```
$ openssl req -new -key pihole.key -subj "/CN=pihole.home.local" -out pihole.csr
```
Use a common name of your choice. You can also use a real DNS name if your network has it own DNS server. If you want to publicly expose the service however it is advisable that you use a proper cluster issuer such as cert-manager for LetsEncrypt, but again this is out of the scope of this guide
- Finally create the certificate:
```
$ openssl x509 -req -in pihole.csr -days 3600 -signkey pihole.key -out pihole.crt
```
Import the certificate and the private key as a Kubernetes `Secret` of type `tls`:
```
$ kubectl create secret tls pihole-tls --cert=pihole.crt --key=pihole.key
```

Now its time to create the ingress:
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: pihole-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: pihole-svc
          servicePort: 80
  tls:
  - hosts:
    - pihole.home.local
    secretName: pihole-tls
```
- Notice that at the `tls[].hosts` we are using the same CN we defined on the CSR creation 
- Notice that we are using the `nginx.ingress.kubernetes.io/force-ssl-redirect: "true"` annotation to force the ingress the redirect to https even if the client requests an http connection

Now create the ingress:
```
$ kubectl create -f pihole-ingress.yaml
```

Verify that the ingress has been created:

```
$ kubectl get ingress pihole-ingress
NAME             CLASS    HOSTS   ADDRESS         PORTS     AGE
pihole-ingress   <none>   *       192.168.1.220   80, 443   3m
```

If you aim to connect within your private network and you are using a custom DNS name, you'll probably need to add an entry to the `/etc/hosts` of your device:
```
...
192.168.1.220 pihole.home.local
...
```

That should do it! Point your browser to `https://<dns-name-of-your-choice>/admin` to access the admin dashboard, and edit your network settings on your devices to point to the IP of the pod as the primary DNS resolver, and happy ad-free browsing!