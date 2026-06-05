# Modul 32 – Aufräumen: Lernpfad 5 abschließen

## Was du in Lernpfad 5 gebaut hast

| Ressource | Typ | Modul | Kosten/Monat |
|-----------|-----|-------|-------------|
| `rg-container` | Resource Group | alle | kostenlos |
| `acrtraining{XXXX}` | Azure Container Registry (Basic) | 27 | ca. 5 € |
| Container Instances | ACI | 28 | ca. 0,05–1 € (nach Nutzung) |
| `env-container` | Container Apps Environment | 29 | kostenlos (nur Apps kosten) |
| `flask-container-app`, `meine-container-app` | Container Apps | 29 | ca. 0 € bei Scale-to-zero |
| `log-container` | Log Analytics Workspace | 29 | ca. 2 € (erste 5 GB/Monat kostenlos) |
| `aks-training` | AKS Cluster (1x Standard_B2s Node) | 30, 31 | ca. **35 €/Monat** ⚠️ |

!!! warning "AKS als Hauptkostentreiber"
    Der AKS-Cluster ist der teuerste Bestandteil dieses Lernpfads – **auch wenn du ihn nicht nutzt, läuft der Node als VM**. Stelle sicher dass du den Cluster löschst!

---

## Reihenfolge beim Aufräumen

AKS zuerst löschen (dauert am längsten), dann ACR und Container Apps:

---

## Schritt 1: AKS-Cluster löschen

```bash
az aks delete \
  --resource-group rg-container \
  --name aks-training \
  --yes \
  --no-wait
```

`--no-wait` gibt die Shell sofort zurück – der Löschvorgang läuft im Hintergrund (5–10 Minuten).

Während AKS gelöscht wird, weiter mit den nächsten Schritten.

---

## Schritt 2: Container Apps löschen

```bash
# Alle Container Apps in der Resource Group
az containerapp list \
  --resource-group rg-container \
  --query "[].name" -o tsv | \
  while read app; do
    az containerapp delete \
      --name $app \
      --resource-group rg-container \
      --yes
    echo "Gelöscht: $app"
  done

# Container Apps Environment löschen
az containerapp env delete \
  --name env-container \
  --resource-group rg-container \
  --yes
```

---

## Schritt 3: Container Instances löschen

```bash
# Alle ACI in der Resource Group
az container list \
  --resource-group rg-container \
  --query "[].name" -o tsv | \
  while read c; do
    az container delete \
      --name $c \
      --resource-group rg-container \
      --yes
  done
```

---

## Schritt 4: Log Analytics Workspace löschen

```bash
az monitor log-analytics workspace delete \
  --resource-group rg-container \
  --workspace-name log-container \
  --yes
```

---

## Schritt 5: Azure Container Registry löschen

```bash
ACR_NAME=$(az acr list --resource-group rg-container --query "[0].name" -o tsv)
az acr delete \
  --name $ACR_NAME \
  --resource-group rg-container \
  --yes
```

---

## Schritt 6: Alles auf einmal (Alternative)

Wenn du einfach alles löschen willst:

```bash
az group delete --name rg-container --yes --no-wait
```

Diese eine Zeile löscht alle Ressourcen in der Gruppe asynchron.

---

## kubectl-Kontext aufräumen

Nach dem Cluster-Löschen bleibt ein veralteter Eintrag in `~/.kube/config`:

```bash
# Alle konfigurierten Cluster anzeigen
kubectl config get-clusters

# AKS-Kontext entfernen
kubectl config delete-cluster aks-training
kubectl config delete-context aks-training
```

---

## Was du in Lernpfad 5 gelernt hast

| Modul | Thema | Schlüsselkonzept |
|-------|-------|-----------------|
| 26 – Docker | Images, Container, Dockerfile | Layer, `docker build`, `docker run` |
| 27 – ACR | Private Container Registry | `az acr build`, `docker push` |
| 28 – ACI | Container ohne Server | Pay-per-second, Batch-Jobs |
| 29 – Container Apps | Serverless mit Autoscale | Revisionen, KEDA, Scale-to-zero |
| 30 – AKS Überblick | Kubernetes-Konzepte | Pod, Deployment, Service, Namespace |
| 31 – AKS Workload | Deployen und skalieren | HPA, Rolling Update, Rollback |

### Konzepte die du jetzt kennst

**Container-Grundlagen:**
- ✅ Ein Image ist die Bauanleitung, ein Container ist die laufende Instanz
- ✅ Dockerfile: `FROM` → `COPY` → `RUN` → `CMD`
- ✅ Layer-Caching: Abhängigkeiten vor App-Code im Dockerfile

**Azure-Container-Dienste:**
- ✅ ACR: private Image-Registry direkt in Azure
- ✅ ACI: schnellste Art einen Container zu starten – ideal für Einzel-Jobs
- ✅ Container Apps: serverlos, Autoscaling, kein K8s-Wissen nötig
- ✅ AKS: volle Kubernetes-Kontrolle, für komplexe Workloads

**Kubernetes:**
- ✅ Deployment = gewünschter Zustand, Kubernetes sorgt dafür
- ✅ Service = stabiler Endpunkt vor wechselnden Pods
- ✅ HPA skaliert automatisch nach CPU oder Custom Metrics
- ✅ Rolling Update: Update ohne Downtime, Rollback bei Fehlern

---

## Lernpfad 5 – Checkliste

- [ ] Dockerfile geschrieben, Image gebaut und als Container lokal gestartet
- [ ] Image in Azure Container Registry gepusht
- [ ] Container als ACI gestartet und per URL erreichbar gemacht
- [ ] Container App mit Scale-to-zero konfiguriert
- [ ] AKS-Cluster mit `kubectl` verbunden
- [ ] Deployment + Service per YAML deployt
- [ ] HPA erstellt und Skalierungsverhalten beobachtet

---

Weiter zu [Lernpfad 6 – DevOps & IaC: Modul 33 – Azure DevOps](modul-33-devops.md) →

!!! info "Lernpfad 6 – DevOps & IaC"
    In Lernpfad 6 geht es um Automatisierung: Wie werden Azure-Ressourcen per Code beschrieben (Bicep, Terraform) und wie werden Apps automatisch gebaut und deployt (GitHub Actions, Azure DevOps)?
