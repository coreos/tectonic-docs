# Reconfigure etcd to use client authentication

Tectonic Installer 1.8.9-tectonic.1 and 1.7.14-tectonic.1 and earlier deployed etcd members without requiring client authentication.

These etcd nodes were configured to use client auth, and were provisioned with the necessary TLS material to enable auth.

To enable client auth on these etcd nodes, simply modify the etcd unit to include the flags, and restart the unit.

First, add the required flags:

```
$ sudo vim /etc/systemd/system/etcd-member.service.d/40-etcd-cluster.conf
```

Add the following flags:

```
--trusted-ca-file=/etc/ssl/etcd/ca.crt \
--client-cert-auth=true \
```

Then, reload and restart the unit:

```
$ sudo systemctl daemon-reload
$ sudo systemctl restart etcd-member
```

Finally, confirm that each instance rejects unauthenticated requests:

```
$ ETCDCTL_API=3 etcdctl \ --cacert /etc/ssl/etcd/ca.crt \ --endpoints=https://127.0.0.1:2379 \ get / --prefix --keys-only
```
