# Modul 30 – Azure Kubernetes Service: Überblick und Konzepte

## Lernziele

Nach diesem Modul kannst du:

- Die Kernkonzepte von Kubernetes (Node, Pod, Deployment, Service) erklären
- Den Unterschied zwischen selbst verwaltetem Kubernetes und AKS beschreiben
- Einen AKS-Cluster erstellen und `kubectl` verbinden
- Die wichtigsten `kubectl`-Befehle für Diagnose anwenden
- AKS-Komponenten im Azure Portal finden und verstehen

---

## Hintergrund: Was ist Kubernetes?

**On-Prem-Vergleich:** Stell dir vor du hast 20 Container die auf verschiedene Server verteilt werden müssen. Manuell auf jedem Server `docker run` aufzurufen, ausgefallene Container neuzustarten und Updates einzuspielen wäre mühsam. **Kubernetes** (kurz: K8s) automatisiert genau das: es verteilt Container auf Nodes, startet sie bei Ausfall neu und skaliert sie auf Wunsch.

**Kubernetes-Grundbegriffe:**

| Begriff | Bedeutung | On-Prem-Analog |
|---------|-----------|---------------|
| **Node** | VM/Server im Cluster | Server im Rechenzentrum |
| **Pod** | Kleinste Einheit: 1+ Container | Prozess auf einem Server |
| **Deployment** | Deklarative Beschreibung: "3 Replicas meiner App" | App-Konfiguration |
| **Service** | Stabiler Endpunkt für Pods (Load Balancer intern) | DNS-Alias / VIP |
| **Namespace** | Logische Trennung innerhalb des Clusters | Organisationseinheit / OU |
| **ConfigMap** | Konfiguration als Kubernetes-Objekt | .config-Datei |
| **Secret** | Sensible Konfiguration (verschlüsselt) | Windows Credential Manager |

**Kubernetes-Architektur:**

```
Control Plane (von Azure verwaltet bei AKS)
├── API Server       ← kubectl spricht hier
├── Scheduler        ← entscheidet welcher Node
├── etcd             ← Cluster-Zustand (Datenbank)
└── Controller Manager

Worker Nodes (deine VMs)
├── kubelet          ← kommuniziert mit Control Plane
├── container runtime (containerd)
└── Pods             ← deine Container
```

Bei **AKS** verwaltet Azure den Control Plane kostenlos. Du zahlst nur für die Worker Nodes (VMs).

---

## AKS-Cluster erstellen

!!! warning "Kosten beachten"
    AKS-Worker Nodes sind VMs die dauerhaft laufen (solange der Cluster existiert). Ein `Standard_B2s` (2 vCPU, 4 GB RAM) kostet ca. **30–35 €/Monat**. Lösche den Cluster nach dem Training!

### Im Portal

1. Suche nach **Kubernetes services** → **+ Create** → **Kubernetes cluster**
2. Tab **Basics**:

| Feld | Wert |
|------|------|
| Resource group | `rg-container` |
| Cluster name | `aks-training` |
| Region | `West Europe` |
| Kubernetes version | neueste stabile |
| Automatic upgrade | `Patch (recommended)` |

3. Tab **Node pools**: Standardpool auf 1 Node setzen, Größe `Standard_B2s`
4. Tab **Integrations**: Container registry → deine ACR auswählen (verbindet automatisch per Managed Identity)
5. **Review + create** → **Create** (dauert 3–5 Minuten)

### Per CLI (Cloud Shell)

```bash
# Cluster erstellen (dauert 3–5 Minuten)
az aks create \
  --resource-group rg-container \
  --name aks-training \
  --node-count 1 \
  --node-vm-size Standard_B2s \
  --generate-ssh-keys \
  --enable-managed-identity \
  --attach-acr $(az acr list --resource-group rg-container --query "[0].name" -o tsv)
```

- `--node-count 1` – ein Worker Node (für Training ausreichend)
- `--attach-acr` – verbindet AKS direkt mit der ACR (Managed Identity bekommt AcrPull-Rolle)

### kubectl installieren und verbinden

```bash
# kubectl installieren (falls nicht vorhanden)
az aks install-cli

# Verbindungsdaten für kubectl holen
az aks get-credentials \
  --resource-group rg-container \
  --name aks-training

# Verbindung testen
kubectl get nodes
```

Ausgabe zeigt deinen Node mit Status `Ready`.

---

## Cluster erkunden

```bash
# Alle Namespaces
kubectl get namespaces

# System-Pods (Kubernetes-Infrastruktur)
kubectl get pods --namespace kube-system

# Alles im default-Namespace
kubectl get all
```

### Im Azure Portal

1. Suche nach **Kubernetes services**
2. Klicke auf `aks-training`

Wichtige Bereiche:
- **Overview**: Cluster-Status, Kubernetes-Version, Node-Anzahl
- **Workloads**: Deployments, Pods, ReplicaSets
- **Services and ingresses**: Externe Endpunkte
- **Configuration**: ConfigMaps und Secrets
- **Node pools**: Worker-Node-Gruppen verwalten
- **Insights**: Monitoring (CPU, RAM, Container-Logs)

---

## Kubernetes-Objekte: YAML verstehen

Kubernetes-Ressourcen werden als YAML definiert. Grundstruktur:

```yaml
apiVersion: apps/v1      # API-Gruppe
kind: Deployment          # Objekttyp
metadata:
  name: mein-deployment
  namespace: default
spec:                     # gewünschter Zustand
  replicas: 2
  selector:
    matchLabels:
      app: mein-webserver
  template:               # Pod-Vorlage
    metadata:
      labels:
        app: mein-webserver
    spec:
      containers:
      - name: webserver
        image: mein-webserver:1.0
        ports:
        - containerPort: 8080
```

**Wichtigste `kubectl`-Befehle:**

```bash
# Ressourcen erstellen/aktualisieren
kubectl apply -f datei.yaml

# Ressourcen anzeigen
kubectl get pods
kubectl get deployments
kubectl get services

# Details zu einer Ressource
kubectl describe pod <POD-NAME>

# Logs eines Pods
kubectl logs <POD-NAME>

# Interaktive Shell in einen Pod
kubectl exec -it <POD-NAME> -- /bin/sh

# Ressource löschen
kubectl delete -f datei.yaml
```

---

## Labels und Selektoren

Labels sind Key-Value-Paare auf Kubernetes-Objekten. Services finden ihre Pods über Label-Selektoren:

```yaml
# Service sucht Pods mit Label "app: mein-webserver"
selector:
  app: mein-webserver
```

```bash
# Alle Pods mit einem bestimmten Label
kubectl get pods -l app=mein-webserver
```

---

## Namespaces: Cluster aufteilen

Namespaces trennen logisch unterschiedliche Teams/Umgebungen:

```bash
# Namespace erstellen
kubectl create namespace dev
kubectl create namespace prod

# Ressourcen in einem Namespace erstellen
kubectl apply -f deployment.yaml --namespace dev

# Ressourcen in einem Namespace anzeigen
kubectl get pods --namespace dev
```

---

## AKS vs. Container Apps: Entscheidungshilfe

| | Container Apps | AKS |
|--|---------------|-----|
| Kubernetes-Kenntnisse nötig | Nein | Ja |
| Kontrolle über Cluster | Keine | Volle |
| Autoscaling | Eingebaut | HPA/KEDA konfigurieren |
| Networking | Einfach | Komplex (CNI, Ingress) |
| Kosten | Pay-per-use | Fix (VM-Kosten) |
| Kubernetes-APIs direkt | Nein | Ja |
| Custom Operators | Nein | Ja |

**Faustregel:** Für die meisten Web-Apps und APIs ist **Container Apps** einfacher und günstiger. **AKS** wenn du volle Kubernetes-Kontrolle brauchst oder bereits Kubernetes-Know-how hast.

---

## Challenge

!!! question "Challenge: Cluster erkunden"
    Verbinde dich mit dem AKS-Cluster und beantworte per `kubectl`:
    
    1. Welche Kubernetes-Version läuft auf dem Node?
    2. Wie viele Pods laufen im `kube-system` Namespace?
    3. Welche Services gibt es im `default` Namespace?
    4. Was zeigt `kubectl cluster-info`?

??? success "Hinweis"
    ```bash
    kubectl get nodes -o wide           # Kubernetes-Version
    kubectl get pods -n kube-system     # System-Pods zählen
    kubectl get services                # Services im default-Namespace
    kubectl cluster-info                # Control Plane URL und Services
    ```

---

Weiter zu [Modul 31 – AKS: Ersten Workload deployen und skalieren](modul-31-aks-workload.md) →
