# Modul 31 – AKS: Ersten Workload deployen und skalieren

## Lernziele

Nach diesem Modul kannst du:

- Ein Deployment und einen Service per YAML in AKS deployen
- Pods skalieren (manuell und automatisch per HPA)
- Ein Rolling Update durchführen ohne Downtime
- Einen Ingress Controller einrichten und externe Anfragen routen
- Persistente Volumes für zustandsbehaftete Apps verwenden

---

## Vorbereitung: Cluster verbinden

```bash
az aks get-credentials \
  --resource-group rg-container \
  --name aks-training

kubectl get nodes  # sollte Ready zeigen
```

---

## Deployment erstellen

Erstelle die Datei `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webserver-deployment
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webserver
  template:
    metadata:
      labels:
        app: webserver
    spec:
      containers:
      - name: webserver
        image: DEIN-ACR-NAME.azurecr.io/mein-webserver:1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
```

Ersetze `DEIN-ACR-NAME` mit dem tatsächlichen ACR-Namen:

```bash
ACR_NAME=$(az acr list --resource-group rg-container --query "[0].name" -o tsv)
sed -i "s/DEIN-ACR-NAME/$ACR_NAME/" deployment.yaml
```

Deployen:

```bash
kubectl apply -f deployment.yaml

# Pods beobachten
kubectl get pods -w  # -w = watch (Ctrl+C zum Beenden)
```

!!! info "Resource Requests und Limits"
    - `requests`: Mindestressourcen die der Scheduler reserviert (für Scheduling-Entscheidungen)
    - `limits`: Maximale Ressourcen – Container wird gedrosselt/neugestartet wenn überschritten
    - `100m` = 100 Millicores = 0,1 CPU-Kern

---

## Service erstellen: Pods von außen erreichbar machen

Erstelle `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webserver-service
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: webserver
  ports:
  - port: 80
    targetPort: 8080
```

```bash
kubectl apply -f service.yaml

# Externe IP abwarten (dauert ~1 Minute)
kubectl get service webserver-service -w
```

Sobald eine externe IP erscheint (Spalte `EXTERNAL-IP`): öffne `http://{EXTERNE-IP}` im Browser.

!!! info "Service Typen"
    | Type | Beschreibung |
    |------|-------------|
    | `ClusterIP` | Nur intern erreichbar (default) |
    | `NodePort` | Über Node-IP + Port erreichbar |
    | `LoadBalancer` | Azure Load Balancer + öffentliche IP |

---

## Manuelles Skalieren

```bash
# Auf 4 Replicas hochskalieren
kubectl scale deployment webserver-deployment --replicas=4

# Pods beobachten
kubectl get pods

# Zurück auf 2
kubectl scale deployment webserver-deployment --replicas=2
```

---

## Horizontaler Pod Autoscaler (HPA)

Der HPA skaliert automatisch basierend auf CPU-Last:

Erstelle `hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webserver-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webserver-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

```bash
kubectl apply -f hpa.yaml

# HPA-Status
kubectl get hpa
```

Bei über 50% CPU-Auslastung im Durchschnitt skaliert der HPA automatisch neue Pods hoch.

!!! tip "Metrics Server"
    Der HPA braucht Metriken vom Metrics Server. Bei AKS ist dieser automatisch installiert – du kannst sofort loslegen.

---

## Rolling Update: Update ohne Downtime

```bash
# Neues Image deployen (Rolling Update)
kubectl set image deployment/webserver-deployment \
  webserver=$ACR_NAME.azurecr.io/mein-webserver:2.0

# Update-Status verfolgen
kubectl rollout status deployment/webserver-deployment
```

Kubernetes tauscht Pods schrittweise aus (standardmäßig: 25% gleichzeitig) – kein Downtime.

### Rollback bei Fehlern

```bash
# Letztes funktionierendes Deployment wiederherstellen
kubectl rollout undo deployment/webserver-deployment

# Bestimmte Revision
kubectl rollout history deployment/webserver-deployment
kubectl rollout undo deployment/webserver-deployment --to-revision=2
```

---

## ConfigMap: Konfiguration externalisieren

Konfigurationswerte gehören nicht ins Image – sie kommen per ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webserver-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
```

Im Deployment referenzieren:

```yaml
envFrom:
- configMapRef:
    name: webserver-config
```

---

## Kubernetes Secret: Passwörter sicher

```bash
# Secret aus Literalen erstellen
kubectl create secret generic db-credentials \
  --from-literal=username=dbadmin \
  --from-literal=password=SuperGeheim123
```

Im Deployment:

```yaml
env:
- name: DB_USER
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: username
- name: DB_PASS
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password
```

!!! warning "Kubernetes Secrets sind Base64, nicht verschlüsselt"
    Standard-Kubernetes Secrets sind nur Base64-kodiert – wer Zugriff auf etcd hat, kann sie lesen. In Produktion: Azure Key Vault über die **Secret Store CSI Driver**-Integration nutzen.

---

## Logs und Diagnose

```bash
# Logs eines Pods
kubectl logs <POD-NAME>

# Logs aller Pods eines Deployments
kubectl logs -l app=webserver

# Logs eines abgestürzten Containers (vorheriger Start)
kubectl logs <POD-NAME> --previous

# Shell im Pod
kubectl exec -it <POD-NAME> -- /bin/sh

# Pod-Details (Events, Status, Ressourcen)
kubectl describe pod <POD-NAME>
```

---

## Challenge

!!! question "Challenge: Flask-App auf AKS"
    Deploye die `flask-app:1.0` aus der ACR auf AKS:
    
    1. Erstelle Deployment (2 Replicas, Port 5000) und Service (LoadBalancer, Port 80 → 5000)
    2. Warte bis eine externe IP vergeben ist
    3. Erstelle einen HPA (min: 2, max: 8, CPU-Target: 60%)
    4. Skaliere manuell auf 4 Replicas und beobachte die Pods

??? success "Hinweis"
    Deployment:
    ```yaml
    containers:
    - name: flask
      image: DEIN-ACR.azurecr.io/flask-app:1.0
      ports:
      - containerPort: 5000
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "300m"
          memory: "256Mi"
    ```
    
    Service:
    ```yaml
    spec:
      type: LoadBalancer
      selector:
        app: flask
      ports:
      - port: 80
        targetPort: 5000
    ```
    
    HPA:
    ```bash
    kubectl autoscale deployment flask-deployment \
      --min=2 --max=8 --cpu-percent=60
    ```

---

Weiter zu [Modul 32 – Aufräumen Lernpfad 5](modul-32-aufräumen.md) →
