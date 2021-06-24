# grafana-prometheus-template

This template deploys Prometheus and Grafana for monitoring pods running in the same Rahti namespace. 

Prometheus is a systems and service monitoring system. It collects metrics from configured targets at given intervals, evaluates rule expressions, displays the results, and can trigger alerts when specified conditions are observed. Grafana is open source visualization and analytics software. It allows you to query, visualize, alert on, and explore your metrics no matter where they are stored. It provides tools to turn time-series database (TSDB) data into graphs and visualizations.

You can read more about Prometheus [here](https://prometheus.io/docs/introduction/overview/) and about Grafana [here](https://grafana.com/docs/grafana/latest/getting-started/).

## Using the template with oc CLI tool

Login to Rahti
```bash
$ oc login https://rahti.csc.fi:8443 --token=<token from Rahti>
```

Create a new project
```bash
$ oc new-project <your Rahti project name> --description="csc_project:<your CSC project name>"
```

Deploy Prometheus and Grafana using the template
```
$ oc new-app -f <path/to/template> -p PARAM1=Value1 -p PARAM2=Value2
```


### Parameters to be supplied

(Note: memory requests and limits should only be changed after checking project quota to avoid errors)
| Parameter                 | Description                                                | Default value                                       |
| ------------------------- | ---------------------------------------------------------- | --------------------------------------------------- |
| NAMESPACE                 | The namespace Prometheus and Grafana are being deployed to | (no default)                                        |
| PROMETHEUS_IMAGE          | The location of the Prometheus image                       | prom/prometheus:v2.27.1                             |
| PROMETHEUS_RETENTION_TIME | Storage retention time for Prometheus                      | 15d                                                 |
| PROMETHEUS_VOLUMESIZE     | Size of the persistent volume for prometheus               | 10Gi                                                |
| PROMETHEUS_LIMITMEMORY    | Memory limit for Prometheus                                | 4G                                                  |
| PROMETHEUS_REQMEMORY      | Requested memory for Prometheus                            | 4G                                                  |
| BASIC_AUTH_USERNAME       | Username for prometheus basic auth                         | admin                                               |
| BASIC_AUTH_PASSWORD       | Password for prometheus basic auth                         | (no default, generated automatically if left empty) |
| GRAFANA_IMAGE             | The location of the Grafana image                          | grafana/grafana:7.5.7                               |
| GRAFANA_ADMIN_USERNAME    | Username for the Grafana admin user                        | admin                                               |
| GRAFANA_ADMIN_PASSWORD    | Password for the Grafana admin user                        | (no default, generated automatically if left empty) |
| GRAFANA_VOLUMESIZE        | Size of the persistent volume for Grafana                  | 100Mi                                               |
| GRAFANA_LIMITMEMORY       | Memory limit for Grafana                                   | 1Gi                                                 |
| GRAFANA_REQMEMORY         | Requested memory for Grafana                               | 512Mi                                               |

## Monitoring applications

### Pods running in the same namespace

Prometheus is configured by default to be able to monitor pods running in the same namespace.

You need to add the following to `metadata.annotations` in the deployment configs of the pods you want to monitor:
- `prometheus.io/scrape: 'true'`
- `prometheus.io/path: <path>` if you need to scrape metrics from a path other than `/metrics`
- `prometheus.io/port: <port>` if you need to use a port other than the pod's declared ports


### Apps outside of the namespace

Add new monitoring targets to the ConfigMap `prometheus-config` under `scrape_configs` for each application you want to monitor. After editing the ConfigMap, redeploy Prometheus to see the changes. More about configuring monitoring targets can be found [here](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config).


#### No authentication
If your application uses no authentication, you can add a new monitoring target with:

```yaml
- job_name: '<job name>'
  static_configs:
  - targets: ['<URL>']
    labels:
      <label name>: '<label value>'
  metrics_path: '<path/to/metrics (default: /metrics)>'
  scheme: '<scheme (default: http)>'
```


#### Basic auth
If your application is using basic auth, you can use:

```yaml
- job_name: '<job name>'
  static_configs:
  - targets: ['<URL>']
    labels:
      <label name>: '<label value>'
  metrics_path: '<path/to/metrics (default: /metrics)>'
  scheme: '<scheme (default: http)>'
  basic_auth:
    username: '<username>'
    password: '<password>'
```
It is also possible to store the password in a file. Use `password_file: '<path/to/file>'` and mount a secret to the Prometheus deployment with the file as data.


#### Token authorization
If your application uses token authorization with HTTP headers, you can use:

```yaml
- job_name: '<job name>'
  static_configs:
  - targets: ['<URL>']
    labels:
      <label name>: '<label value>'
  metrics_path: '<path/to/metrics (default: /metrics)>'
  scheme: '<scheme (default: http)>'
  authorization:
    type: '<type (default: Bearer)>'
    credentials: '<secret>'
```
It is also possible to store the credentials in a file. Use `credentials_file: '<path/to/file>'` and mount a secret to the Prometheus deployment with the file as data.

#### IP whitelisting on Rahti
If your applications are running in Rahti, it is advisable to use IP whitelisting to allow connections to your monitoring targets from Rahti only. 

On Rahti, you can do this with:
```bash
$ oc annotate route <route_name> haproxy.router.openshift.io/ip_whitelist='193.167.189.25'
```

On Rahti-int you can do this with:
```bash
$ oc annotate route <route_name> haproxy.router.openshift.io/ip_whitelist='193.166.25.134'
```

**Note: Doing this will allow connections from all Rahti namespaces. Contact Rahti admins if you want a specific egress IP for your namespace.**
