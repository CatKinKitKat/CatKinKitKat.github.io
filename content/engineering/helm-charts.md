+++
title = "Helm Charts: Packaging Kubernetes Apps Without Losing Your Mind"
date = 2025-07-05
[taxonomies]
tags = ["kubernetes", "devops", "infrastructure"]
+++

The first Helm chart I wrote was a disaster. It had 47 template files, a values.yaml that was 600 lines long, and Go template logic so deeply nested that I needed a debugger to understand what YAML it would produce. I was so proud of how "flexible" it was. Nobody else could use it. Including me, two weeks later.

Helm charts should be boring. They should be readable. They should produce predictable output. Everything else is vanity.

## Chart Structure

A Helm chart is a directory:

```
order-service/
  Chart.yaml          # metadata
  values.yaml         # default config
  templates/
    deployment.yaml
    service.yaml
    configmap.yaml
    ingress.yaml
    _helpers.tpl      # shared template functions
    NOTES.txt         # post-install message
```

That's it. Five or six template files for a typical microservice. If you have more than ten, question every one of them.

`Chart.yaml` is metadata:

```yaml
apiVersion: v2
name: order-service
description: Order management service
version: 1.0.0      # chart version
appVersion: "2.3.1"  # application version
```

Keep `version` and `appVersion` separate. The chart version changes when the deployment configuration changes. The app version changes when the application changes. They're independent.

## Values Templating: Less Is More

Here's a values.yaml that parameterizes what actually varies between environments:

```yaml
replicaCount: 2

image:
  repository: myregistry.azurecr.io/order-service
  tag: "latest"
  pullPolicy: IfNotPresent

resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "1000m"

config:
  databaseUrl: "jdbc:postgresql://db:5432/orders"
  logLevel: "INFO"

ingress:
  enabled: true
  host: orders.myapp.com
```

And the deployment template:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "order-service.fullname" . }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "order-service.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "order-service.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 8080
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

The template is readable. You can look at it and know what it produces. The Go template syntax is ugly, but it's predictable.

The common mistake: parameterizing everything. I've seen charts where the container port is a value, the protocol is a value, the service type has five conditional branches, and there's a nested loop generating environment variables from a map of maps. Stop. If the port is always 8080, hardcode 8080. Parameterize things that change between environments. Hardcode things that don't.

## The _helpers.tpl File

This is where you define reusable template snippets:

```yaml
{{- define "order-service.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{- define "order-service.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "order-service.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

Standard Kubernetes labels. Every chart needs them. Use `helm create` to generate the boilerplate and modify from there.

## Dependencies

Your chart can depend on other charts. The classic example: your service needs PostgreSQL.

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "12.1.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

```yaml
# values.yaml
postgresql:
  enabled: true
  auth:
    database: orders
    username: orderservice
  primary:
    persistence:
      size: 10Gi
```

Run `helm dependency update` and the PostgreSQL chart gets pulled into your `charts/` directory. When you install your chart, PostgreSQL gets installed alongside your service.

This is great for development and testing. For production, I prefer managing databases separately from application charts. You don't want `helm upgrade order-service` to accidentally trigger a PostgreSQL upgrade.

## Creating and Publishing Charts

Package your chart:

```bash
helm package order-service/
# creates order-service-1.0.0.tgz
```

For internal distribution, push to an OCI registry (your existing container registry works):

```bash
helm push order-service-1.0.0.tgz oci://myregistry.azurecr.io/helm
```

Install from the registry:

```bash
helm install order-service oci://myregistry.azurecr.io/helm/order-service --version 1.0.0
```

Using your container registry as a Helm chart registry is the simplest approach. No ChartMuseum server, no GitHub Pages hosting. ACR, ECR, and GCR all support OCI artifacts.

## Testing Charts

Always template before applying:

```bash
helm template order-service ./order-service -f values-staging.yaml
```

This renders the templates without installing anything. Pipe it through `kubectl apply --dry-run=server` for validation.

For automated testing, use `helm unittest`:

```yaml
# tests/deployment_test.yaml
suite: test deployment
templates:
  - deployment.yaml
tests:
  - it: should set correct replica count
    set:
      replicaCount: 5
    asserts:
      - equal:
          path: spec.replicas
          value: 5
```

Unit testing Helm charts feels over-engineered until you've deployed a chart that rendered invalid YAML because of an indent error in a conditional block. Then it feels essential.

## Helm vs Kubernetes Operators

Helm installs and configures resources. It's a package manager. An Operator is a custom controller that actively manages the lifecycle of a resource - it watches, reconciles, handles upgrades, backups, failover.

Use Helm for stateless applications that just need a Deployment, Service, and ConfigMap. The chart installs them and Kubernetes handles the rest.

Use Operators for stateful, complex workloads: databases, message brokers, search clusters. These need active management - handling leader election, scaling, backups, version upgrades. Helm can install an Operator, but Helm itself can't do what an Operator does.

The overlap: Operators often use Helm charts internally. The Operator manages the lifecycle, and the Helm chart defines the resources. They're complementary tools.

## The Honest Advice

Start with `helm create`. Delete the files you don't need. Keep the templates straightforward. Resist the urge to make everything configurable - you're not building a generic chart for the community, you're deploying your specific service.

If your chart's values.yaml needs a README to explain it, your chart is too complex. Simplify until the values are self-explanatory. Your future self will thank you at 2 AM when something is broken and you need to understand what a chart value actually does.
