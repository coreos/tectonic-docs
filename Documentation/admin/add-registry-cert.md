# Add Registry Certificate to Tectonic Worker Nodes

Many enterprises use self-signed certificates to protect container registries. On kubernetes this can present a problem as docker requires that self-signed certificates be nested under the `/etc/docker/certs.d` on every host that will pull from a registry with a self-signed certificate.

This restriction can be solved with a DaemonSet that copies the `ca.crt` file to the needed directory on each host.

A secret will be mounted into the DaemonSet as a file and then copied. This secret must include the base64 encoded contents of the root CA (pem format) used to sign the container registry certs.

```bash
$ base64 -w 0 rootCA.pem
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURWekNDQWorZ0F3SUJBZ0lKQUxRd3FGRWVpakdyTUEwR0NTcUdTSWIzRFFFQkN3VUFNRUl4Q3pBSkJnTlYKQkFZVEFsaFlNUlV3[...]
```


#### registry-secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: registry-ca
  namespace: kube-system
type: Opaque
data:
  registry-ca: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURWekNDQWorZ0F3SUJBZ0lKQUxRd3FGRWVpakdyTUEwR0NTcUdTSWIzRFFFQkN3VUFNRUl4Q3pBSkJnTlYKQkFZVEFsaFlNUlV3[...]
```

Use kubectl to create the `registry-ca` secret:

```
kubectl create -f registry-secret.yaml
```

The following DaemonSet mounts the CA as the file `/home/core/registry-ca` and then copies this file to the `/etc/docker/certs.d/reg.example.com/ca.crt`.

Replace `reg.example.com` with the hostname of your container registry.

#### registry-ca-ds.yaml

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: registry-ca
  namespace: kube-system
  labels:
    k8s-app: registry-ca
spec:
  template:
    metadata:
      labels:
        name: registry-ca
    spec:
      containers:
      - name: registry-ca
        image: busybox
        command: [ 'sh' ]
        args: [ '-c', 'cp /home/core/registry-ca /etc/docker/certs.d/reg.example.com/ca.crt && exec tail -f /dev/null' ]
        volumeMounts:
        - name: etc-docker
          mountPath: /etc/docker/certs.d/reg.example.com
        - name: ca-cert
          mountPath: /home/core
      terminationGracePeriodSeconds: 30
      volumes:
      - name: etc-docker
        hostPath:
          path: /etc/docker/certs.d/reg.example.com
      - name: ca-cert
        secret:
          secretName: registry-ca
```

Use kubectl to create the `registry-ca` DaemonSet:

```
kubectl create -f registry-ca-ds.yaml
```

Checking for success can be accomplished by deploying a Pod or DaemonSet that pulls from the container registry.

If new nodes will be added to the cluster in an automatic fashion the registry-ca daemonset should be left running so new worker nodes receive the certificate. Otherwise the Daemonset can be be removed:

```
kubectl -n kube-system delete ds registry-ca
```
