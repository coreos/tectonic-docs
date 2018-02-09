<br>
<div class="alert alert-info" role="alert">
<i class="fa fa-exclamation-triangle"></i><b> Note:</b> This documentation is for an alpha feature. For questions and feedback on the Multi-cluster Alpha program, email <a href="mailto:tectonic-alpha-feedback@coreos.com">tectonic-alpha-feedback@coreos.com</a>.
</div>

# Multi-cluster user access policies

Cluster Policies define a set of Namespaces and RBAC rules that exist on all clusters that match the Policy's label query. They provide a quick and consistent means for cluster administrators to configure access to a set of clusters for a new team or application. Policies are stored in a single directory which allows for easy security auditing.

The format of Cluster Policies will likely change as the alpha program progresses. Details about migration steps will be provided as they become necessary.

## Create an identity federation access policy

Create a policy which grants admin access to the cluster for a defined LDAP/SAML group:

```
kind: ClusterPolicy
apiVersion: multicluster.coreos.com/v1
metadata:
  name: ldap-group-admin
spec:
  selector:
    cloud: aws
  authorization:
    clusterBindings:
    - clusterRole: admin
      groups: ["Operations"]
```

Submit the new policy to the directory cluster, and watch it get synced to the replica. First, be sure to reset kubeconfig to the correct cluster:

```
export KUBECONFIG=/path/to/kubeconfig
kubectl get nodes

kubectl apply -f sample-policies/ldap-group-admin.yaml
clusterpolicy "ldap-group-admin" created
```

After a few seconds, a Cluster Role Binding will be created on all matching clusters.

## Use multiple namespaces to manage team access

This example creates a set of namespaces and access rules for an engineering team that produces a web API. It does the following:
* Creates a production namespace
  * Grants view access to the support group and a specific user, so they can assess status of the environment
  * Grants edit access to all of the API developers in order to debug and troubleshoot
  * Grants admin access to the team lead to manage the namespace
* Creates a test namespace
  * Grants admin access to all API developers

Example policy to create these namespaces, and grant appropriate access:

```
kind: ClusterPolicy
apiVersion: multicluster.coreos.com/v1
metadata:
  name: web-api
spec:
  selector:
    cloud: aws
  namespaces:
  - name: "api-prod"
    authorization:
      bindings:
      - clusterRole: view
        users: ["random-user"]
        groups: ["SupportTeam"]
      - clusterRole: edit
        groups: ["APIDevelopers"]
      - clusterRole: admin
        users: ["joe-team-lead"]
  - name: "api-test"
    authorization:
      bindings:
      - clusterRole: admin
        groups: ["APIDevelopers"]
```

Submit it to the directory cluster and watch the namespaces and role bindings get created on matching replica clusters:

```
kubectl apply -f sample-policies/namespaces-with-binding.yaml
clusterpolicy "web-api" created
```
