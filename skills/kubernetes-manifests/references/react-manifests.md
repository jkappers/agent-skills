# React / Vite Frontend Kubernetes Manifests

Kubernetes manifest patterns specific to React applications served via NGINX.

## Table of Contents

- [Core Challenge](#core-challenge)
- [NGINX Configuration for SPAs](#nginx-configuration-for-spas)
- [Runtime Environment Variables](#runtime-environment-variables)
- [Health Checks](#health-checks)
- [Non-Root NGINX Security](#non-root-nginx-security)
- [Content Security Policy](#content-security-policy)
- [Resource Recommendations](#resource-recommendations)
- [Complete Example](#complete-example)

## Core Challenge

React applications compile to static files that run in the browser:
- No server-side runtime — the browser is the runtime
- Environment variables baked at build time (`import.meta.env`)
- NGINX serves static files — lightweight, fast, requires SPA routing config

## NGINX Configuration for SPAs

```nginx
server {
    listen 8080;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    gzip on;
    gzip_vary on;
    gzip_min_length 256;
    gzip_comp_level 5;
    gzip_types text/plain text/css text/javascript application/javascript
               application/json application/xml image/svg+xml font/woff2;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    location = /index.html {
        expires -1;
        add_header Cache-Control "no-store, no-cache, must-revalidate";
    }

    location = /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

| Directive | Purpose |
|-----------|---------|
| `try_files $uri $uri/ /index.html` | Enable client-side routing (React Router) |
| `expires 1y` for assets | Aggressive caching for hashed files |
| `expires -1` for index.html | Never cache the entry point |
| `gzip on` | 60-80% response size reduction |

## Runtime Environment Variables

Since env vars are baked at build time, use window object injection for runtime config:

**1. Create `public/config.js` template:**

```javascript
window.RUNTIME_CONFIG = {
  API_URL: "${API_URL}",
  FEATURE_FLAGS: "${FEATURE_FLAGS}"
};
```

**2. Add to `index.html`:**

```html
<script src="/config.js"></script>
```

**3. Create `docker-entrypoint.sh`:**

```bash
#!/bin/sh
envsubst < /usr/share/nginx/html/config.js.template > /usr/share/nginx/html/config.js
exec nginx -g 'daemon off;'
```

**4. Access in React:**

```javascript
const apiUrl = window.RUNTIME_CONFIG?.API_URL || 'http://localhost:3000';
```

**5. Kubernetes ConfigMap:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
data:
  API_URL: "https://api.production.example.com"
  FEATURE_FLAGS: '{"newDashboard": true}'
```

> **Never store secrets in frontend configuration.** Environment variables are embedded in client-side JavaScript visible to anyone inspecting the app.

## Health Checks

NGINX serving static files has no application-level health distinction. Use the same `/health` endpoint for liveness and readiness:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

## Non-Root NGINX Security

Use `nginxinc/nginx-unprivileged:1.27-alpine`:
- Runs as non-root user (UID 101)
- Default port: 8080 (not 80)
- PID file at `/tmp/nginx.pid`
- Works with `readOnlyRootFilesystem: true`

Mount emptyDir volumes for writable paths:

```yaml
volumeMounts:
- name: tmp
  mountPath: /tmp
- name: cache
  mountPath: /var/cache/nginx
volumes:
- name: tmp
  emptyDir: {}
- name: cache
  emptyDir: {}
```

## Content Security Policy

Add CSP headers via NGINX or Ingress annotation:

```nginx
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self' https://api.example.com;" always;
```

Or via Ingress:

```yaml
annotations:
  nginx.ingress.kubernetes.io/configuration-snippet: |
    add_header Content-Security-Policy "default-src 'self'; script-src 'self';" always;
```

## Resource Recommendations

Frontend containers serving static files are lightweight:

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
```

## Complete Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app.kubernetes.io/name: frontend
    app.kubernetes.io/component: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        runAsGroup: 101
        fsGroup: 101
      containers:
      - name: frontend
        image: myregistry/frontend:1.0.0
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: API_URL
          valueFrom:
            configMapKeyRef:
              name: frontend-config
              key: API_URL
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
              - ALL
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache/nginx
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: frontend
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: frontend-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```
