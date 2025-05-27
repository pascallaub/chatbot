# AI Chatbot mit Ollama und Open WebUI

Dieses Projekt stellt eine vollständige AI-Chatbot-Lösung mit Ollama (Backend) und Open WebUI (Frontend) in Kubernetes bereit.

## Architektur

- **Ollama**: AI-Model-Backend für lokale LLM-Ausführung
- **Open WebUI**: Benutzerfreundliche Weboberfläche für Chat-Interaktionen
- **Kubernetes**: Container-Orchestrierung mit Ingress-Controller
- **NGINX Ingress**: Load Balancing und Routing

## Voraussetzungen

- Docker Desktop mit aktiviertem Kubernetes
- kubectl konfiguriert für lokalen Cluster
- NGINX Ingress Controller installiert
- Mindestens 8GB RAM verfügbar
- Windows mit administrativen Rechten (für hosts-Datei)

## Installation

### 1. Repository klonen und Verzeichnis wechseln

```bash
git clone <repository-url>
cd chatbot/kubernetes
```

### 2. NGINX Ingress Controller installieren (falls nicht vorhanden)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```

### 3. Alle Kubernetes-Ressourcen bereitstellen

```bash
# ConfigMap für WebUI-Konfiguration
kubectl apply -f webui.configmap.yaml

# Ollama Backend
kubectl apply -f ollama.deployment.yaml
kubectl apply -f ollama.service.yaml

# Open WebUI Frontend
kubectl apply -f webui.deployment.yaml
kubectl apply -f webui.service.yaml

# Ingress für externen Zugriff
kubectl apply -f ai-chat.ingress.yaml
```

### 4. Hosts-Datei konfigurieren

**PowerShell als Administrator ausführen:**

```powershell
Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value "127.0.0.1 myapp.local"
```

### 5. Deployment-Status überwachen

```bash
# Pods prüfen
kubectl get pods

# Services prüfen
kubectl get services

# Ingress prüfen
kubectl get ingress

# Logs verfolgen
kubectl logs -f -l app=ollama
kubectl logs -f -l app=webui
```

## AI-Modell installieren

### 1. In Ollama-Pod einsteigen

```bash
kubectl exec -it $(kubectl get pod -l app=ollama -o jsonpath='{.items[0].metadata.name}') -- /bin/bash
```

### 2. Modell herunterladen

```bash
# Kleines, schnelles Modell für Tests
ollama pull phi3:mini

# Oder größeres, leistungsfähigeres Modell
ollama pull llama3.2:1b

# Verfügbare Modelle auflisten
ollama list
```

## Zugriff auf die Anwendung

### Option 1: Über Ingress (empfohlen)
- URL: http://myapp.local
- Automatisches Load Balancing
- Produktionsähnliche Konfiguration

### Option 2: Port-Forward (Entwicklung)
```bash
kubectl port-forward service/webui-frontend 8080:8080
```
- URL: http://localhost:8080

## Verwendung

1. **Weboberfläche öffnen**: http://myapp.local
2. **Modell auswählen**: In der WebUI das geladene Modell auswählen
3. **Chat starten**: Fragen eingeben und AI-Antworten erhalten
4. **Einstellungen**: Über Settings → Models weitere Modelle verwalten

## Troubleshooting

### Pod startet nicht (OOMKilled)
```bash
# Resource-Limits in webui.deployment.yaml erhöhen
# Dann neu deployen:
kubectl apply -f webui.deployment.yaml
```

### 502 Bad Gateway
```bash
# Pod-Status prüfen
kubectl get pods
kubectl describe pod <pod-name>

# Logs prüfen
kubectl logs <pod-name>
```

### WebUI kann keine Modelle laden
```bash
# Ollama-Verbindung testen
kubectl exec -it $(kubectl get pod -l app=webui -o jsonpath='{.items[0].metadata.name}') -- curl http://ollama-backend:11434/api/version

# ConfigMap prüfen
kubectl get configmap webui-config -o yaml
```

### Ingress funktioniert nicht
```bash
# Ingress-Controller prüfen
kubectl get pods -n ingress-nginx

# Hosts-Datei prüfen
cat C:\Windows\System32\drivers\etc\hosts | findstr myapp.local
```

## Konfiguration

### Resource-Limits anpassen

**Für mehr Performance (webui.deployment.yaml):**
```yaml
resources:
  requests:
    memory: "4Gi"
    cpu: "2"
  limits:
    memory: "6Gi"
    cpu: "3"
```

### Ollama-API-URL ändern (webui.configmap.yaml)
```yaml
data:
  OLLAMA_API_BASE_URL: "http://ollama-backend:11434"
  OLLAMA_BASE_URL: "http://ollama-backend:11434"
```

## Deinstallation

```bash
# Alle Ressourcen entfernen
kubectl delete -f ai-chat.ingress.yaml
kubectl delete -f webui.service.yaml
kubectl delete -f webui.deployment.yaml
kubectl delete -f webui.configmap.yaml
kubectl delete -f ollama.service.yaml
kubectl delete -f ollama.deployment.yaml

# Hosts-Eintrag entfernen (PowerShell als Administrator)
(Get-Content C:\Windows\System32\drivers\etc\hosts) | Where-Object {$_ -notmatch "myapp.local"} | Set-Content C:\Windows\System32\drivers\etc\hosts
```

## Verfügbare AI-Modelle

| Modell | Größe | Empfehlung |
|--------|-------|------------|
| `phi3:mini` | ~2GB | Schnell, gut für Tests |
| `llama3.2:1b` | ~1GB | Sehr schnell, kompakt |
| `llama3.2:3b` | ~2GB | Ausgewogen |
| `llama3.1:8b` | ~4.7GB | Hochwertige Antworten |

## Support

Bei Problemen:
1. Logs der betroffenen Pods prüfen
2. Resource-Verfügbarkeit überprüfen
3. Netzwerk-Konnektivität zwischen Pods testen
4. ConfigMap-Konfiguration validieren

## Lizenz

Dieses Projekt nutzt Open-Source-Komponenten:
- [Ollama](https://ollama.ai) - MIT License
- [Open WebUI](https://github.com/open-webui/open-webui) - MIT License