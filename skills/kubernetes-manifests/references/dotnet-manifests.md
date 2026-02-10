# .NET Core / ASP.NET Core Kubernetes Manifests

Kubernetes manifest patterns specific to .NET Core and ASP.NET Core applications.

## Table of Contents

- [Health Checks](#health-checks)
- [GC Configuration](#gc-configuration)
- [Graceful Shutdown](#graceful-shutdown)
- [Configuration Management](#configuration-management)
- [Kestrel Port Configuration](#kestrel-port-configuration)
- [Docker Image Selection](#docker-image-selection)
- [Logging](#logging)
- [Complete Example](#complete-example)

## Health Checks

ASP.NET Core has built-in health check support. Configure separate endpoints:

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddSqlServer(connectionString, name: "database")
    .AddRedis(redisConnectionString, name: "redis");

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // Liveness: just check app is running
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

app.MapHealthChecks("/health/startup", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("startup")
});
```

Manifest probe configuration:

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 15
  timeoutSeconds: 5
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 10
startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

Use the [AspNetCore.Diagnostics.HealthChecks](https://github.com/xabaril/aspnetcore.diagnostics.healthchecks) library for pre-built checks (SQL Server, PostgreSQL, Redis, RabbitMQ).

## GC Configuration

Server GC (ASP.NET Core default) assumes abundant memory and causes OOM kills in containers.

| Mode | Memory Usage | Best For |
|------|-------------|----------|
| Server GC (default) | Higher | High-throughput, dedicated servers |
| Workstation GC | Lower | Resource-constrained containers |

For memory-constrained containers, switch to Workstation GC:

```yaml
env:
- name: DOTNET_gcServer
  value: "0"
```

.NET Core 3.0+ automatically respects container memory limits:
- Default GC heap size: 75% of container memory limit
- CPU count derived from cgroup CPU limits
- More logical CPUs = more GC heaps = higher memory usage

## Graceful Shutdown

ASP.NET Core stops accepting new requests on `SIGTERM`. Default shutdown timeout: 30 seconds.

**NGINX Ingress timing issue**: There is a delay between Kubernetes removing a pod from Service endpoints and NGINX updating its configuration. Use a `preStop` hook:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "10"]
terminationGracePeriodSeconds: 45
```

`terminationGracePeriodSeconds` must exceed `preStop` delay + shutdown timeout.

## Configuration Management

ASP.NET Core configuration priority (highest to lowest):
1. Command-line arguments
2. Environment variables
3. `appsettings.{Environment}.json`
4. `appsettings.json`

Use double underscores for nested settings in environment variables:

```yaml
env:
- name: ASPNETCORE_ENVIRONMENT
  value: "Production"
- name: ConnectionStrings__DefaultConnection
  valueFrom:
    secretKeyRef:
      name: db-secrets
      key: connection-string
- name: AppSettings__ApiUrl
  value: "https://api.internal.svc.cluster.local"
```

Mount appsettings overrides via ConfigMap:

```yaml
volumeMounts:
- name: config
  mountPath: /app/appsettings.Production.json
  subPath: appsettings.Production.json
  readOnly: true
volumes:
- name: config
  configMap:
    name: myapp-config
```

**Never** store secrets in appsettings.json, Docker images, or plain environment variables. Use Kubernetes Secrets mounted as volumes or external secret managers.

## Kestrel Port Configuration

```yaml
env:
- name: ASPNETCORE_URLS
  value: "http://+:8080"
```

When running as non-root (required), use ports above 1024.

## Docker Image Selection

| Image Type | Approximate Size |
|------------|-----------------|
| SDK (build only) | ~800MB |
| ASP.NET Runtime | ~220MB |
| ASP.NET Chiseled | ~110MB |
| With Native AOT | ~50-80MB |

Use chiseled/distroless images for production:
- `mcr.microsoft.com/dotnet/aspnet:8.0-jammy-chiseled` — Minimal Ubuntu
- `mcr.microsoft.com/dotnet/aspnet:8.0-jammy-chiseled-extra` — Includes icu and tzdata
- No shell, no package manager, non-root by default
- Use exec form only: `ENTRYPOINT ["dotnet", "MyApp.dll"]`

## Logging

Output structured JSON to stdout for log aggregators:

```csharp
builder.Host.UseSerilog((context, config) =>
{
    config
        .ReadFrom.Configuration(context.Configuration)
        .Enrich.FromLogContext()
        .Enrich.WithProperty("Application", "MyApp")
        .WriteTo.Console(new JsonFormatter());
});
```

Override log levels via environment variables:

```yaml
env:
- name: Serilog__MinimumLevel__Default
  value: "Warning"
- name: Serilog__MinimumLevel__Override__Microsoft
  value: "Warning"
```

Avoid file-based logging in containers. Use correlation IDs for distributed tracing.

## Complete Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1654
        runAsGroup: 1654
        fsGroup: 1654
      containers:
      - name: myapp
        image: myregistry/myapp:1.0.0
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ASPNETCORE_URLS
          value: "http://+:8080"
        - name: DOTNET_EnableDiagnostics
          value: "0"
        - name: ConnectionStrings__DefaultConnection
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: db-connection
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
              - ALL
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8080
          failureThreshold: 30
          periodSeconds: 10
        lifecycle:
          preStop:
            exec:
              command: ["sleep", "10"]
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
      terminationGracePeriodSeconds: 45
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: myapp
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
```
