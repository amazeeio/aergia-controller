# Aergia

> In Greek mythology, Aergia is the personification of sloth, idleness, indolence and laziness

Aergia is a controller that can be used to scale deployments from zero when a request is made to an ingress with a zero scaled deployment.

This controller replaces the ingress-nginx default backend with this custom backend.

This backend is designed to serve generic error handling for any http error. The backend can also leverage [custom errors](https://kubernetes.github.io/ingress-nginx/user-guide/custom-errors/), which can be used to check the kubernetes api to see if the namespace needs to be scaled from zero.

## Usage

An environment can be force idled, force scaled, or unidled using labels on the namespace. All actions still respect the label selectors, but forced actions will bypass any hits checks

### Force Idled
To force idle a namespace, you can label the namespace using `idling.amazee.io/force-idled=true`. This will cause the environment to be immediately scaled down, but the next request to the ingress in the namespace will unidle the namespace

### Force Scaled
To force scale a namespace, you can label the namespace using `idling.amazee.io/force-scaled=true`. This will cause the environment to be immediately scaled down, but the next request to the ingress in the namespace will *NOT* unidle the namespace. A a deployment will be required to unidle this namespace

### Unidle
To unidle a namespace, you can label the namespace using `idling.amazee.io/unidle=true`. This will cause the environment to be scaled back up to its previous state.

### Namespace Idling Overrides
If you want to change a namespaces interval check times outside of the globally applied intervals, the following annotations can be added to the namespace
* `idling.amazee.io/prometheus-interval` - set this to the time interval for prometheus checks, the format must be in [30m|4h|1h30m](https://pkg.go.dev/time#ParseDuration) notation
* `idling.amazee.io/pod-interval` - set this to the time interval for pod uptime checks, the format must be in [30m|4h|1h30m](https://pkg.go.dev/time#ParseDuration) notation

### IP Allow/Block Lists
It is possible to add global IP allow and block lists, the helm chart will have support for handling this creation
* allowing IP addresses via `/lists/allowedips` file which is a single line per entry of ip address to allow
* blocking IP addresses via `/lists/blockedips` file which is a single line per entry of ip address to block

There are also annotations that can be added to specific `Kind: Ingress` objects that allow for ip allow or blocking.
* `idling.amazee.io/ip-allow-list` - a comma separated list of ip addresses to allow, will be checked against x-forward-for, but if true-client-ip is provided it will prefer this.
* `idling.amazee.io/ip-block-list` - a comma separated list of ip addresses to allow, will be checked against x-forward-for, but if true-client-ip is provided it will prefer this.

### UserAgent Allow/Block Lists
It is possible to add global UserAgent allow and block lists, the helm chart will have support for handling this creation
* allowing user agents via a `/lists/allowedagents` file which is a single line per entry of useragents or regex patterns to match against. These must be `go` based regular expressions.
* blocking user agents via a `/lists/blockedagents` file which is a single line per entry of useragents or regex patterns to match against. These must be `go` based regular expressions.

There are also annotations that can be added to specific `Kind: Ingress` objects that allow for user agent allow or blocking.
* `idling.amazee.io/allowed-agents` - a comma separated list of user agents or regex patterns to allow.
* `idling.amazee.io/blocked-agents` - a comma separated list of user agents or regex patterns to block.

## Change the default templates

By using the environment variable `ERROR_FILES_PATH`, and pointing to a location that contains the three templates `error.html`, `forced.html`, and `unidle.html`, you can change what is shown to the end user.

This could be done using a configmap and volume mount to any directory, then update the `ERROR_FILES_PATH` to this directory.

# Installation

Install via helm (https://github.com/amazeeio/charts/tree/main/charts/aergia)

## Custom templates
If installing via helm, you can use this YAML in your values.yaml file and define the templates there.

> See `www/error.html`, `www/forced.html`, and `www/unidle.html` for inspiration

```
templates:
  enabled: false
  error: |
    {{define "base"}}
    <html>
    <body>
    {{ .ErrorCode }} {{ .ErrorMessage }}
    </body>
    </html>
    {{end}}
  unidle: |
    {{define "base"}}
    <html>
    <head>
    <meta http-equiv="refresh" content="{{ .RefreshInterval }}">
    </head>
    <body>
    {{ .ErrorCode }} {{ .ErrorMessage }}
    </body>
    </html>
    {{end}}
  forced: |
    {{define "base"}}
    <html>
    <head>
    </head>
    <body>
    {{ .ErrorCode }} {{ .ErrorMessage }}
    </body>
    </html>
    {{end}}
```

## Prometheus
The idler uses prometheus to check if there has been hits to the ingress in the last defined interval, it only checks status codes of 200.
By default it will talk to a prometheus in cluster `http://monitoring-kube-prometheus-prometheus.monitoring.svc:9090` but this is adjustable with a flag (and via helm values).

### Requirements
One of the requirements of using prometheus is the ability to query for ingress-nginx requests using this metric `nginx_ingress_controller_requests`

You need to ensure that your ingress-nginx controller is scraped for this metric or else the idler will assume there have been 0 hits and idle the environment without hesitation.

An example `ServiceMonitor` is found in this repo under `test-resources/ingress-servicemonitor.yaml`
