# TLS Certificate Rotation in Tectonic

Start by reviewing the general [TLS documentation][tls-certs] and the [TLS topology][tls-topology] for Tectonic to identify the various certificates in the cluster.

We will be using the [CFSSL][cfssl-util] utility to view and manage the certificates, which may be downloaded from [https://pkg.cfssl.org/][cfssl-package].

This guide will examine rotating the certificates for the following components.

  * [Kubernetes API Server](#kubernetes-api-server)
  * [Tectonic Ingress Controller](#tectonic-ingress-controller)
  * [Kubelet](#kubelet)
  * [Tectonic Identity](#tectonic-identity)
  * [etcd](#etcd)

## Kubernetes API Server

The api-server component in Tectonic is run as a DaemonSet, which results in a Pod running on every master node. The TLS certificate that is used by the api-server is stored as a Kubernetes Secret and mounted into each api-server Pod using a Volume.

First, view the information for the cluster to get the API server public hostname:

```
kubectl cluster-info
```

In this example, the name of the cluster is `demo`. Replace this with the name of your actual cluster.

```
Kubernetes master is running at https://demo-api.coreos.com:443
Heapster is running at https://demo-api.coreos.com:443/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://demo-api.coreos.com:443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
...
```

Set a variable that will hold the hostname for the Kubernetes master nodes without the protocol or port number:

```
export API_HOSTNAME=demo-api.coreos.com
```

Now, view the information for the existing certificate using the CFSSL utility:

```
cfssl certinfo -domain ${API_HOSTNAME}
```

You can either obtain the new certificates from a trusted certificate authority (CA) or generate new self-signed certificates based on your own CA.

In this example, we will generate new self-signed certificates using the CA created during the Tectonic installation process.

Create a Certificate Signing Request (CSR) for the api-server. The following is the minimum required for the CSR.

```
cat > apiserver-csr.json <<EOF
{
  "CN": "kube-apiserver",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "kube-master"
    }
  ]
}
EOF
```

Next, generate the new api-server certificate and private key. Set a variable that contains the path to the CA certificate and private key that will be used for the new api-server certificate.

```
export CA_PATH=<PATH-TO-YOUR-CA-CERTIFICATE>
```

Then, set a variable that will hold the IP address for the Kubernetes API service within the cluster. This is usually the first available IP address from the Service CIDR range that was set up when the Tectonic cluster was provisioned. The default is `10.3.0.1` in Tectonic. Verify this by looking at the SANs listed in the existing certificate viewed in the first step above.

```
export SERVICE_ADDRESS=10.3.0.1
```

Finally, the `API_HOSTNAME` variable contains the DNS name for the API endpoint for the cluster that was previously set.

Generate the new certificate and private key using the CFSSL utility.

```
cfssl gencert \
  -ca=${CA_PATH}/ca.crt \
  -ca-key=${CA_PATH}/ca.key \
  -hostname=${API_HOSTNAME},kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster.local,${SERVICE_ADDRESS} \
  apiserver-csr.json | cfssljson -bare apiserver
```

This will generate several files in the current directory.

```
apiserver-key.pem   # new private key
apiserver.csr       # CSR
apiserver.pem       # new certificate
```

Verify the information for the new certificate.

```
cfssl certinfo -cert apiserver.pem
```

Now that the new certificate and private key have been generated, update the Secret in Kubernetes.

```
kubectl patch secret kube-apiserver -n kube-system -p "{\"data\":{\"apiserver.crt\":\"$(base64 -w 0 apiserver.pem)\", \"apiserver.key\":\"$(base64 -w 0 apiserver-key.pem)\"}}"
```

Once the Secret has been updated, move on to listing all of the api-server pods.

```
kubectl get pods -n kube-system -l k8s-app=kube-apiserver
```

You should see the same number of api-server pods as master nodes in the cluster.

```
NAME                   READY     STATUS    RESTARTS   AGE
kube-apiserver-ptqrj   1/1       Running   0          41m
kube-apiserver-w87b7   1/1       Running   0          41m
```

Now, go through and delete each api-server Pod, one at a time. Because the api-server component is deployed as a DaemonSet, a new pod will be created automatically using the new certificate and key.

__NOTE:__ Do not attempt this approach if you have only a single master node! In that case, deleting the lone api-server pod will render the cluster unable to function.

```
kubectl delete pod -n kube-system <POD_NAME>
```

Wait for the new Pod to be created before moving on to the next Pod. When all of the Pods have been processed, verify that the new certificate is being used.

```
cfssl certinfo -domain ${API_HOSTNAME}
```

## Tectonic Ingress Controller

The Tectonic ingress controller is responsible for securely routing traffic to the Tectonic console and identity components. The TLS certificate that is used by the controller is stored as a Kubernetes Secret and mounted into each tectonic-ingress Pod using a Volume.

First, set a `CLUSTER_HOSTNAME` variable that will hold the hostname for the Kubernetes cluster without the protocol or port number. This hostname is comprised of the `tectonic_cluster_name` and `tectonic_base_domain` variables that were defined as part of the Tectonic installation process.

In this example, the name of the cluster is `demo`. Replace this with the name of your cluster.

```
export CLUSTER_HOSTNAME=demo.coreos.com
```

Next, view the information for the existing certificate using the CFSSL utility.

```
cfssl certinfo -domain ${CLUSTER_HOSTNAME}
```

Create a CSR for the tectonic-ingress-controller component. The following is the minimum required for the CSR.

```
cat > ingress-csr.json <<EOF
{
  "CN": "${CLUSTER_HOSTNAME}",
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF
```

Next, generate the new tectonic-ingress-controller certificate and private key. Set a `CA_PATH` variable that contains the path to the CA certificate and private key that will be used for the new certificate.

```
export CA_PATH=<PATH-TO-YOUR-CA-CERTIFICATE>
```

Generate the new certificate and private key using the CFSSL utility.

```
cfssl gencert \
  -ca=${CA_PATH}/ca.crt \
  -ca-key=${CA_PATH}/ca.key \
  -hostname=${CLUSTER_HOSTNAME} \
  ingress-csr.json | cfssljson -bare ingress
```

This will generate several files in the current directory.

```
ingress-key.pem   # new private key
ingress.csr       # CSR
ingress.pem       # new certificate
```

Verify the information for the new certificate.

```
cfssl certinfo -cert ingress.pem
```

Now that the new certificate and private key have been generated, update the Secret in Kubernetes.

```
kubectl patch secret tectonic-ingress-tls-secret -n tectonic-system -p "{\"data\":{\"tls.crt\":\"$(base64 -w 0 ingress.pem)\", \"tls.key\":\"$(base64 -w 0 ingress-key.pem)\"}}"
```

Once the Secret has been updated, the tectonic-ingress-controller component should pick up the change immediately. Verify that the new certificate is being used using the CFSSL utility.

```
cfssl certinfo -domain ${CLUSTER_HOSTNAME}
```

## Kubelet

The kubelet is responsible for maintaining a set of containers on a particular host. In addition, the kubelet exposes an HTTPS endpoint for health monitoring. In Tectonic, the kubelet is run on every master and worker node in the cluster as a systemd unit.

Use the CFSSL utility to view the information for the kubelet certificate from a machine in the cluster with access to the node running the kubelet. By default, the kubelet listens on port `10250`.

```
cfssl certinfo -domain <NODE IP ADDRESS>:10250
```

Connect to one of the nodes in the cluster to verify that the kubelet service is active and running.

```
ssh core@<NODE IP ADDRESS>
sudo systemctl status kubelet
```

By default, the certificate and key that the kubelet uses to secure its interface are generated automatically when the kubelet starts and are placed in `/var/lib/kubelet/pki`.

```
ls -lAh /var/lib/kubelet/pki
kubelet.crt   # certificate
kubelet.key   # private key
```

Because the kubelet automatically generates these files the first time it starts, rotating the certificate is as easy as removing the old certificate and private key, then restarting the kubelet.

```
sudo rm /var/lib/kubelet/pki/kubelet.crt
sudo rm /var/lib/kubelet/pki/kubelet.key
sudo systemctl restart kubelet
```

Once this has completed, the new files will appear.

```
ls -lAh /var/lib/kubelet/pki
```

Use the CFSSL utility from a remote machine to verify the new certificate.

```
cfssl certinfo -domain <NODE IP ADDRESS>:10250
```

Repeat this process for each master and worker node in the cluster.

## Tectonic Identity

The Tectonic Identity service is not exposed externally. To verify the certificate used, run a Pod in the cluster.

First, output some of the information for the Secret and Service
resources used by Tectonic Identity.

```
kubectl get svc -n tectonic-system tectonic-identity-api
```

You should see the information for the service listed. This service is used by
the Tectonic Console when users log in.

```
NAME                     TYPE         CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
tectonic-identity-api    ClusterIP    10.3.36.187    <none>        5557/TCP   44m
```

Next, view the certificate that is currently used by the service, which
is stored as a Secret.

```
kubectl get secret tectonic-identity-grpc-server-secret -n tectonic-system -o jsonpath={.data.tls-cert} | base64 -D | cfssl certinfo -cert -
```

You should see the secret displayed.

Now, create a simple Pod that will be used to verify that the certificate used by tectonic-identity is the one specified in the Secret.

```
kubectl run certcheck -it --rm --restart Never --image alpine -n tectonic-system
```

After a few seconds, the command prompt for the running Pod should be displayed. Install the OpenSSL package to verify the certificate.

```
apk update && apk add openssl
```

Use the `openssl` utility to verify the gRPC server certificate.

```
openssl s_client -connect tectonic-identity-api:5557
```

Compare the output for the value stored in the Secret to the
certificate retrieved using `openssl` to verify that they match.

### Server certificate

In this example, we will generate new self-signed certificates using the CA
created during the Tectonic installation process.

Create a Certificate Signing Request (CSR) for the tectonic-identity service.
The following is the minimum required for the CSR.

```
cat > identity-csr.json <<EOF
{
  "CN": "tectonic-identity-api.tectonic-system.svc.cluster.local",
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF
```

Next, generate the new tectonic-identity certificate and private
key. Set a variable that contains the path to the CA certificate and
private key that will be used for the new tectonic-identity certificate. This is the path to the CA file that was either created during the Tectonic installation process if using self-signed certificates, or provided to you by your trusted CA.

```
export CA_PATH=<PATH-TO-YOUR-CA-CERTIFICATE>
```

Generate the new certificate and private key using the CFSSL utility.

```
cfssl gencert \
  -ca=${CA_PATH}/ca.crt \
  -ca-key=${CA_PATH}/ca.key \
  -hostname=tectonic-identity-api.tectonic-system.svc.cluster.local \
  identity-csr.json | cfssljson -bare identity-server
```

This will generate several files in the current directory.

```
identity-server-key.pem   # new private key
identity-server.csr       # CSR
identity-server.pem       # new certificate
```

Verify the information for the new certificate.

```
cfssl certinfo -cert identity-server.pem
```

Now that the new certificate and private key have been generated, update
the Secret in Kubernetes.

```
kubectl patch secret tectonic-identity-grpc-server-secret -n tectonic-system -p "{\"data\":{\"tls-cert\":\"$(base64 -w 0 identity-server.pem)\", \"tls-key\":\"$(base64 -w 0 identity-server-key.pem)\"}}"
```

Once the Secret has been updated, list all of the tectonic-identity Pods.

```
kubectl get pods -n tectonic-system -l k8s-app=tectonic-identity
```

You should see the same number of api-server Pods as master nodes in the cluster.

```
NAME                      READY     STATUS    RESTARTS   AGE
tectonic-identity-ptqrj   1/1       Running   0          41m
tectonic-identity-w87b7   1/1       Running   0          41m
```

Next, delete each tectonic-identity Pod, one at a time. Because the api-server component is deployed as a Deployment, a new Pod
will be created automatically using the new certificate and key.

```
kubectl delete pod -n tectonic-system <POD_NAME>
```

Wait for the new Pod to be created before moving on to the next. When all of the Pods have been processed, verify that the new certificate is being used.

Once again, create a simple Pod that will be used to verify that the new certificate is used by tectonic-identity.

```
kubectl run certcheck -it --rm --restart Never --image alpine -n tectonic-system
```

After a few seconds, the command prompt for the running Pod should be displayed. Install the OpenSSL package to verify the certificate.

```
apk update && apk add openssl
```

Use the `openssl` utility to verify the gRPC server certificate.

```
openssl s_client -connect tectonic-identity-api:5557
```

Compare the output from the `openssl` utility to the new certificate to verify
everything is working as expected.

### Client certificate

In addition to the server certificate, there is also a client certificate that
is used by Tectonic Console.

```
kubectl get secret tectonic-identity-grpc-client-secret -n tectonic-system -o jsonpath={.data.tls-cert} | base64 -D | cfssl certinfo -cert -
```

You should see the secret displayed.

Generate a new client certificate and private key using the CFSSL utility. Use the same CSR that was used for the server certificate.

```
cfssl gencert \
  -ca=${CA_PATH}/ca.crt \
  -ca-key=${CA_PATH}/ca.key \
  -hostname=tectonic-identity-api.tectonic-system.svc.cluster.local \
  identity-csr.json | cfssljson -bare identity-client
```

This will generate several files in the current directory.

```
identity-client-key.pem   # new private key
identity-client.csr       # CSR
identity-client.pem       # new certificate
```

Verify the information for the new certificate.

```
cfssl certinfo -cert identity-client.pem
```

Now that the new certificate and private key have been generated, update
the Secret in Kubernetes.

```
kubectl patch secret tectonic-identity-grpc-client-secret -n tectonic-system -p "{\"data\":{\"tls-cert\":\"$(base64 -w 0 identity-client.pem)\", \"tls-key\":\"$(base64 -w 0 identity-client-key.pem)\"}}"
```

Tectonic Console should now use the new certificate for subsequent
operations. Log out and back into Tectonic Console to verify everything is working properly.

## etcd

There are several TLS certificates to be managed when administering an etcd
cluster. This example will generate a new CA certificate and use this CA to generate new self-signed certificates for each of the components
in the etcd cluster.

### Verify cluster health

First, verify the current health of the etcd cluster. Connect to one of the etcd members of the cluster using SSH.

```
ssh core@<ETCD MEMBER NODE>
```

etcd clusters should be configured to require client authentication. Therefore, we will need the existing CA certificate, and the client certificate and key for the cluster. These artifacts should be located in the `/etc/ssl/etcd` directory if the Tectonic cluster was set up using self-signed certificates.

```
ls -lAh /etc/ssl/etcd
```

Copy the `tls.zip` file to the machine where the work is being performed.

```
total 68K
-rw-r--r--. 1 root root 1.3K Dec 31  1979 ca.crt
-r--------. 1 root root 1.3K Dec 31  1979 client.crt
-r--------. 1 root root 1.7K Dec 31  1979 client.key
-r--------. 1 etcd etcd 1.6K Dec 31  1979 peer.crt
-r--------. 1 etcd etcd 1.7K Dec 31  1979 peer.key
-r--------. 1 etcd etcd 1.6K Dec 31  1979 server.crt
-r--------. 1 etcd etcd 1.7K Dec 31  1979 server.key
-r--------. 1 root root 8.5K Feb  2 16:32 tls.zip
```

Extract the existing TLS archive somewhere on the machine where the work is being performed.

```
unzip tls.zip
```

Next, set some environment variables to make the commands less
complicated.

Be sure to replace the hostnames for the `ETCDCTL_ENDPOINTS` variable with the
actual hostnames for the cluster. The values listed for some of the variables
below assume that you are working in the same directory where the `tls.zip`
archive was extracted.

```
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=demo-etcd-0.coreos.com:2379,demo-etcd-1.coreos.com:2379,demo-etcd-2.coreos.com:2379
export ETCDCTL_CACERT=$(pwd)/ca.crt
export ETCDCTL_CERT=$(pwd)/client.crt
export ETCDCTL_KEY=$(pwd)/client.key
```

Once the variables are set, use `etcdctl` to verify the health of the cluster.

```
etcdctl endpoint health
```

A message indicating the health for each endpoint in the cluster will be returned.

```
demo-etcd-0.coreos.com:2379 is healthy: successfully committed proposal: took = 2.802157ms
demo-etcd-1.coreos.com:2379 is healthy: successfully committed proposal: took = 2.313969ms
demo-etcd-2.coreos.com:2379 is healthy: successfully committed proposal: took = 2.768635ms
```

Make a note of the leader node in the etcd cluster for use later in this example.

```
etcdctl endpoint status
```

The fifth (5th) column with the Boolean values indicates whether or not the node is the leader for the cluster.

```
demo-etcd-0.coreos.com:2379, 1524e8fe213236a7, 3.1.8, 8.4 MB, false, 98, 80367
demo-etcd-1.coreos.com:2379, 7046411f5ee24eaf, 3.1.8, 8.5 MB, false, 98, 80367
demo-etcd-2.coreos.com:2379, 60f732f378671843, 3.1.8, 8.5 MB, true, 98, 80367
```

### Cluster CA

Once the health of the cluster has been verified, start the process of
generating the new TLS certificates that will be used. If you are not using
self-signed certificates, you can obtain the new certificates from the CA of
your choice and skip to the [Rotate etcd Certificates](#rotate-etcd-certificates) section below.

First, generate the configuration file needed to create the CSR for the CA using the CFSSL utility.

```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "etcd": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

Next, create a Certificate Signing Request (CSR) for the etcd CA. The
following is the minimum required for the CSR.

```
cat > etcd-ca-csr.json <<EOF
{
  "CN": "etcd-ca",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "etcd"
    }
  ]
}
EOF
```

```
cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare etcd-ca
```

This will generate several files in the current directory.

```
etcd-ca-key.pem   # new private key
etcd-ca.csr       # CSR
etcd-ca.pem       # new certificate
```

Verify the information for the new CA certificate.

```
cfssl certinfo -cert etcd-ca.pem
```

Use the CA certificate and key to self-sign new certificates for the
etcd cluster components.


### Peer

The peer certificate is used for etcd-to-etcd member communication.

Create a Certificate Signing Request (CSR) for the etcd peers. The following is the minimum required for the CSR.

```
cat > etcd-peer-csr.json <<EOF
{
  "CN": "etcd",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "etcd"
    }
  ]
}
EOF
```

Set a variable that contains the list of DNS names for each member of the etcd
cluster. These names must be updated to match the DNS names used in the cluster.

```
export ETCD_DNS_NAMES=demo-etcd-0.coreos.com,demo-etcd-1.coreos.com,demo-etcd-2.coreos.com
```

Set a variable that contains the Kubernetes service IP addresses.

```
export SERVICE_ADDRESSES=10.3.0.15,10.3.0.20
```

Generate the new certificate and private key using the CFSSL utility.

```
cfssl gencert \
  -ca=etcd-ca.pem \
  -ca-key=etcd-ca-key.pem \
  -hostname=${ETCD_DNS_NAMES},*.kube-etcd.kube-system.svc.cluster.local,kube-etcd-client.kube-system.svc.cluster.local,${SERVICE_ADDRESSES} \
  etcd-peer-csr.json | cfssljson -bare etcd-peer
```

This will generate several files in the current directory.

```
etcd-peer-key.pem   # new private key
etcd-peer.csr       # CSR
etcd-peer.pem       # new certificate
```

Verify the information for the new certificate.

```
cfssl certinfo -cert etcd-peer.pem
```

### Server

The server certificate is used to secure connections to the etcd server.

Create a Certificate Signing Request (CSR) for the etcd server. The following is the minimum required for the CSR.

```
cat > etcd-server-csr.json <<EOF
{
  "CN": "etcd",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "etcd"
    }
  ]
}
EOF
```

Use the CFSSL utility to generate the new certificate and private key.

```
cfssl gencert \
  -ca=etcd-ca.pem \
  -ca-key=etcd-ca-key.pem \
  -hostname=${ETCD_DNS_NAMES},localhost,*.kube-etcd.kube-system.svc.cluster.local,kube-etcd-client.kube-system.svc.cluster.local,127.0.0.1,${SERVICE_ADDRESSES} \
  etcd-server-csr.json | cfssljson -bare etcd-server
```

This will generate several files in the current directory.

```
etcd-server-key.pem   # new private key
etcd-server.csr       # CSR
etcd-server.pem       # new certificate
```

Verify the information for the new certificate.

```
cfssl certinfo -cert etcd-server.pem
```

### Client

The client certificate is used for "api server"-to-etcd and
"etcd operator"-to-etcd client communication.

Create a Certificate Signing Request (CSR) for etcd clients. The following is the minimum required for the CSR.

```
cat > etcd-client-csr.json <<EOF
{
  "CN": "etcd",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "etcd"
    }
  ]
}
EOF
```

Generate the new certificate and private key using the CFSSL utility. The warning about a missing hosts field may be safely ignored.

```
cfssl gencert \
  -ca=etcd-ca.pem \
  -ca-key=etcd-ca-key.pem \
  -config=ca-config.json \
  -profile=etcd \
  etcd-client-csr.json | cfssljson -bare etcd-client
```

This will generate several files in the current directory.

```
etcd-client-key.pem   # new private key
etcd-client.csr       # CSR
etcd-client.pem       # new certificate
```

Verify the information for the new certificate.

```
cfssl certinfo -cert etcd-client.pem
```

### Rotate etcd certificates

Now that the new certificates have been generated, replace the existing
certificates with the new ones. These steps must be performed in the following
order.

First, replace the etcd client certificate and key that are used
by the api-server to communicate with the etcd backend.

```
kubectl patch secret kube-apiserver -n kube-system -p "{\"data\":{\"etcd-client-ca.crt\":\"$(base64 -w 0 etcd-ca.pem)\", \"etcd-client.crt\":\"$(base64 -w 0 etcd-client.pem)\", \"etcd-client.key\":\"$(base64 -w 0 etcd-client-key.pem)\"}}"
```

Next, prepare the new certificates to be uploaded to each etcd member
node.

Create a temporary directory and rename the files to match what the etcd-member service expects.

```
mkdir tmp
cp etcd-ca.pem tmp/ca.crt
cp etcd-client.pem tmp/client.crt
cp etcd-client-key.pem tmp/client.key
cp etcd-peer.pem tmp/peer.crt
cp etcd-peer-key.pem tmp/peer.key
cp etcd-server.pem tmp/server.crt
cp etcd-server-key.pem tmp/server.key
```

Create an archive to contain the new certificates and set the permissions
appropriately.

```
cd tmp
chmod 400 client.* server.* peer.*
chmod 644 ca.crt
sudo chown 232:232 peer.* server.*
sudo chown 0:0 ca.crt client.*
sudo zip -j tls-new.zip ca.crt client.* peer.* server.*
```

Next, replace the peer and server certificates on each of the etcd member
nodes. Upload the new certificates to each etcd member in the
cluster using SCP or something similar.

```
scp tls-new.zip core@<ETCD MEMBER NODE>:/home/core/tls-new.zip
```

Log into the first etcd member using SSH once the new certificates have been
copied over.

```
ssh core@<ETCD MEMBER NODE>
```

Stop the etcd-member service.

```
sudo systemctl --no-block stop etcd-member
```

The default location for the certificates that are used by the etcd service is
the `/etc/ssl/etcd` directory. Verify this by viewing the systemd unit
file for the etcd-member service.

```
cat /etc/systemd/system/etcd-member.service.d/40-etcd-cluster.conf
```

Make note of the path listed for each of the following CLI flags. Either the new certificates must be named the same, or the unit file for the
etcd-member service must be updated as well.

```
--cert-file
--key-file
--peer-cert-file
--peer-key-file
--peer-trusted-ca
```

View the list of files in the `/etc/ssl/etcd` directory.

```
ls -lAh /etc/ssl/etcd
```

Copy the archive with the new certificates into place and change to the
directory.

```
sudo cp /home/core/tls-new.zip /etc/ssl/etcd/
cd /etc/ssl/etcd/
ls -lAh
```

Remove the old certificates and unzip the archive with the new certificates.

```
sudo rm -f ca.crt client.* peer.* server.* tls.zip
sudo mv tls-new.zip tls.zip
sudo unzip tls.zip
ls -lAh
```

Ensure that the `peer` and `server` certificates are owned by the `etcd` system user.

```
sudo chown etcd: peer.* server.*
ls -lAh
```

Start the etcd-member service and verify that it is operational. You may see
warnings that the certificate authority does not match, this is normal.

```
sudo systemctl --no-block start etcd-member
sudo systemctl status etcd-member
```

Repeat this process on each etcd member node in cluster, saving the etcd leader node for the very last.

Once all of the certificates have been replaced, log into each master
node and restart the api-server container to pick up the changes made to
the Secret for the etcd client certificate and key.

```
ssh core@<MASTER NODE>
```

Get the ID for the api-server container running on the host.

```
docker ps | grep k8s_kube-apiserver
```

You should see a single container listed.

```
857a8a911536  quay.io/coreos/hyperkube@...
```

Restart the api-server container, replacing the ID with the actual value for
the container.

```
docker restart <API SERVER CONTAINER ID>
```

Repeat this process on each master node in the cluster.

Run `kubectl` commands or log into Tectonic Console to verify that everything is functioning as expected.

```
kubectl cluster-info
```


[tls-certs]: tls-certificates.md
[tls-topology]: tls-topology.md
[cfssl-util]: https://blog.cloudflare.com/introducing-cfssl/
[cfssl-package]: https://pkg.cfssl.org/
