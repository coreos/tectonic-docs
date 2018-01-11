# Prometheus Open Cloud Service

Tectonic’s Prometheus Open Cloud Service provides a one-click, fully managed, application monitoring and metrics stack for both operations and applications teams on-top of a Tectonic cluster. Use Prometheus’s Alertmanager to route, email, page, or message teams when something goes wrong with an application or the container infrastructure. Prometheus OCS is:

* **Highly available:** Configure redundancy to ensure important metrics are never missed in production or save resources in development environments by running a single instance.
* **Kubernetes Native:** Use native Kubernetes paradigms like Pod label selectors and automatically track all application containers even as they are upgraded, destroyed, or rescheduled.
* **High performance:** Containers encourage the creation of more services and CoreOS has ensured Prometheus can track all of those services while using minimal resources. (For more information on Prometheus, see the blog post [Prometheus, the backbone of container monitoring, hits 2.0][prom-20].)

Allows you to deploy and manage Prometheus instance into any namespace.

## Deploying Prometheus OCS

Use Tectonic Console to enable the Prometheus OCS for selected namespaces.

By default, objects created using the Prometheus OCS will be labeled `prometheus=k8s`.

Using the Prometheus Open Cloud Service to deploy a Prometheus instance will create the following Kubernetes objects:
* A Prometheus CRD
* A Prometheus Service Monitor
* A Prometheus Stateful Set
* A Prometheus Secret

## Ingesting metrics

Prometheus Open Cloud Service will not enable Ingress for the cluster. First configure Ingress to access the Prometheus UI.

Once enabled, go to *https://{your-cluster-dns}/prometheus* to access Prometheus monitoring for the cluster. Select *Status > Targets* to confirm that Prometheus is correctly configured and ingesting metrics.

<div class="row">
  <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-10 col-sm-offset-1 col-xs-10 col-xs-offset-1">
    <a href="../img/prometheus-targets.png" class="co-m-screenshot">
      <img src="../img/prometheus-targets.png" class="img-responsive">
    </a>
  </div>
</div>

## Using Alertmanager

Use the built-in Tectonic Alertmanager with the Prometheus OCS. Alertmanagers should be shared between Prometheus instances.

For Service discovery, the Prometheus Pod must have permission to access the Kubernetes API in the tectonic-system namespace. Follow the instructions in [Monitoring Applications][monitoring-apps] to create a ClusterRoleBinding to bind the available ClusterRole to an appropriate ServiceAccount. Using Tectonic Console to create a Prometheus instance will automatically generate the required ClusterRoleBinding.

For more information, see [Exposing Prometheus and Alertmanager][exposing-prometheus] and [Alerting][alerting-md].

## Working with Kubernetes Services to monitor your app

Prometheus must have sufficient RBAC permissions to access the Kubernetes cluster.

The app must be instrumented, and expose an HTTP endpoint. Use the [Client Library][client-library] appropriate to your app to expose metrics through an HTTP endpoint.

Then, configure Prometheus to discover these targets.
1. Create a Service selecting the Pods of the deployed app.
2. Create a ServiceMonitor object to select the Service objects to be monitored by the Prometheus server. Use a label-selector to define the objects to be monitored.
3. Configure Prometheus to select the ServiceMonitor.

For more information, see [Application Monitoring][app-monitoring].


[app-monitoring]: https://coreos.com/tectonic/docs/latest/tectonic-prometheus-operator/user-guides/application-monitoring.html
[client-library]: https://prometheus.io/docs/instrumenting/clientlibs/
[exposing-prometheus]:  https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/exposing-prometheus-and-alertmanager.md
[alerting-md]:  https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/alerting.md
[monitoring-apps]: https://coreos.com/tectonic/docs/latest/tectonic-prometheus-operator/user-guides/application-monitoring.html
[prom-20]: https://coreos.com/blog/prometheus-2.0-released
