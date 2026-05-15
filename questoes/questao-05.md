# Questão 05 - Modernizar deployment legado

## Prompt

```markdown
# Before

Considerando o manifesto onde existem problemas em produção:

- Sem alta disponibilidade
- imagem utilizando a versão latest 
- secrets expostas dentro do manifest
- ausência de requests/limits
- ausência de liveness/readiness probes
- ausência de outras boas práticas em operação de Kubernetes

Manifesto legado de entrada:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      containers:
      - name: api
        image: chronos-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          value: "P@ssw0rd2023!"
        - name: JWT_SECRET
          value: "hvt-jwt-prod-secret"
```

# After

Produza uma versão moderna seguindo melhores práticas para ambiente de produção considerando:

1. alta disponibilidade
2. imagem versionada (sem uso de versão latest)
3. segredos fora do manifesto (referência via Secret)
4. resources.requests e resources.limits
5. livenessProbe e readinessProbe
6. demais práticas recomendadas de produção para Deployment Kubernetes

# Bridge

Você é um especialista Kubernetes com foco em produção, segurança e confiabilidade.

- Reescreva o manifesto aplicando as mudanças necessárias de forma explícita preservando name, namespace, labels principais e porta
- Use valores realistas e seguros por padrão.
- Inclua apenas YAML final em blocos separados por --- caso precise de mais de um recurso (ex.: Secret + Deployment).
- Não inclua explicações textuais fora do YAML.

## Modelo

Foi utilizado o modelo `Claude Sonnet 4.6` pela expertise do Claude para com códigos e soluções técnica

## Output

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: chronos-api-secrets
  namespace: production
  labels:
    app: chronos-api
type: Opaque
stringData:
  DB_PASSWORD: "CHANGE_ME_BEFORE_APPLY"
  JWT_SECRET: "CHANGE_ME_BEFORE_APPLY"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  labels:
    app: chronos-api
    version: "1.0.0"
    managed-by: kubectl
spec:
  replicas: 3
  revisionHistoryLimit: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - chronos-api
            topologyKey: kubernetes.io/hostname
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: chronos-api
      terminationGracePeriodSeconds: 60
      containers:
      - name: api
        image: chronos-api:1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: chronos-api-secrets
              key: DB_PASSWORD
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: chronos-api-secrets
              key: JWT_SECRET
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: http
          initialDelaySeconds: 10
          periodSeconds: 10
          failureThreshold: 3
          successThreshold: 1
          timeoutSeconds: 3
        livenessProbe:
          httpGet:
            path: /health/live
            port: http
          initialDelaySeconds: 20
          periodSeconds: 15
          failureThreshold: 3
          successThreshold: 1
          timeoutSeconds: 5
        startupProbe:
          httpGet:
            path: /health/live
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 12
          timeoutSeconds: 3
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: app-cache
          mountPath: /app/cache
      volumes:
      - name: tmp
        emptyDir: {}
      - name: app-cache
        emptyDir: {}
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: chronos-api-pdb
  namespace: production
  labels:
    app: chronos-api
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: chronos-api
```

## Justificativa

É especificado o BEFORE com detalhes do que temos atualmente no deployment legado com lacunas pontos criticos de produção. Já no After definimos o objetivo quanto ao estado do manifesto, trazendo alta disponibilidade, imagem versionada, Secret, requests/limits, liveness/readiness, boas práticas para ambientes kubernetes. 
Já o Bridge conecta os dois estados com instruções operacionais precisas de transformação, ou seja, reescrever o manifesto preservando identidade e contexto aplicando defaults seguros e restringir a saída a YAML final.
