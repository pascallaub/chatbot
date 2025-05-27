# Kubernetes Übung: AI Chat App Deployment

## Ziel der Aufgabe

Nachdem wir die Bausteine von Kubernetes (Pods, Deployments, Services, Ingress, ConfigMaps) und die Interaktion über `kubectl` kennengelernt haben, wird nun eine reale Multi-Container-Anwendung – die AI Chat App – in einem lokalen Kubernetes Cluster deployt. Ziel ist es, die zwei Hauptkomponenten (Ollama Backend, Open WebUI Frontend) als Deployments und Services zu definieren, ihre Verbindung über eine ConfigMap zu konfigurieren und den externen Zugriff über Ingress zu ermöglichen. Diese Aufgabe dient als grosse Übung zur Anwendung aller bisher gelernten K8s-Grundlagen.

## Lernziele dieser Aufgabe

- Eine komplexe Multi-Container-Anwendung in die passenden stateless Kubernetes-Objekte zerlegen (Deployments, Services).
- Services definieren, die interne (ClusterIP) und externe (Ingress) Kommunikation ermöglichen.
- Anwendungskonfiguration über eine ConfigMap bereitstellen und als Umgebungsvariable injizieren.
- Externen Zugriff über Ingress basierend auf einem Hostnamen konfigurieren.
- Das Zusammenspiel aller Objekte (Labels/Selektoren, Referenzen, ConfigMaps) in der Praxis umsetzen.
- Den gesamten Stack mit `kubectl apply -f <directory>` deployen und testen.
- Die Auswirkung des Fehlens von Persistenz für stateful Komponenten (Ollama-Modelle) beobachten und verstehen.

## Anleitung

### Voraussetzungen prüfen & Lokales Cluster starten

1.  **Docker Desktop**: Stelle sicher, dass Docker installiert und läuft.
2.  **Lokales Kubernetes Cluster**: Stelle sicher, dass dein lokales Kubernetes Cluster (Docker Desktop Kubernetes, Minikube oder Kind) läuft und `kubectl` darauf zugreift.
    ```bash
    kubectl get nodes
    ```
3.  **Ingress Controller**: Stelle sicher, dass der Ingress Controller in deinem Cluster installiert oder aktiviert ist. Für Docker Desktop ist er meist standardmässig dabei.
    ```bash
    # Beispiel für NGINX Ingress Controller Installation (falls nicht vorhanden)
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
    # Warte bis der Ingress Controller Pod läuft:
    kubectl get pods -n ingress-nginx
    ```
4.  **Metrics Server**: Stelle sicher, dass der Metrics Server in deinem Cluster installiert ist (nützlich für `kubectl top`).
    ```bash
    # Beispiel für Metrics Server Installation (falls nicht vorhanden)
    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    # Prüfen:
    kubectl top nodes
    ```
5.  **Ressourcen**: Konfiguriere die empfohlenen Ressourcen in Docker Desktop (mind. 12 GB RAM, 6 CPU Cores) für die AI-Modelle.
6.  **Docker Hub Login**: Stelle sicher, dass du bei Docker Hub angemeldet bist (falls nötig, falls Images nicht öffentlich sind).
7.  **Optionale Vorbereitung: Lokale DNS-Konfiguration**: Passe deine lokale Hosts-Datei an, um den Hostnamen `chat.local` auf die IP-Adresse deines Clusters aufzulösen.
    -   **Windows (PowerShell als Administrator)**:
        ```powershell
        Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value "127.0.0.1 chat.local"
        ```
        *(Hinweis: `127.0.0.1` ist für Docker Desktop Ingress auf localhost üblich. Für Minikube `minikube ip` verwenden).*

### Anwendungskomponenten & Ihre K8s Objekte identifizieren

Analysiere die Komponenten der AI Chat App:

-   **Ollama**: Dies ist der AI/LLM Backend-Service, der die grossen Sprachmodelle (AI-Modelle) lädt und Anfragen verarbeitet.
    -   **Stateful oder Stateless?** Ollama speichert heruntergeladene Modelle im Dateisystem (`/root/.ollama`). Daher ist es *stateful*.
    -   **Welches Objekt?** Ideal wäre ein `StatefulSet` mit einem `PersistentVolumeClaim`. Für diese Übung nutzen wir jedoch ein `Deployment`, um die Limitationen ohne Persistenz zu demonstrieren.
-   **Open WebUI**: Dies ist das Web-Frontend für die Chat-Oberfläche.
    -   **Stateful oder Stateless?** Das Frontend selbst ist *stateless*. Es speichert keine kritischen Daten, die einen Neustart nicht überleben würden.
    -   **Welches Objekt?** Ein `Deployment` ist hierfür gut geeignet.

**Zusätzliche Objekte:**
-   **Services**: Jede Komponente (Ollama, WebUI) benötigt einen `Service` vom Typ `ClusterIP` für die interne Erreichbarkeit.
-   **Ingress**: Für den externen Zugriff auf das WebUI Frontend wird ein `Ingress` benötigt.
-   **ConfigMap**: Die URL des Ollama-Backends (`OLLAMA_API_BASE_URL`) wird über eine `ConfigMap` an die WebUI übergeben.

### Kubernetes Manifeste erstellen (YAML)

Erstelle im `kubernetes/` Ordner deines Repositories die folgenden YAML-Dateien:

1.  **`ollama.deployment.yaml`**:
    ```yaml
    # filepath: kubernetes/ollama.deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ollama-backend
      labels:
        app: ollama
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: ollama
      template:
        metadata:
          labels:
            app: ollama
        spec:
          containers:
            - name: ollama
              image: ollama/ollama:latest # Öffentlich verfügbares Image
              ports:
                - containerPort: 11434
              resources: # Wichtig für Scheduling und Performance
                requests:
                  memory: "2Gi"
                  cpu: "1"
                limits:
                  memory: "4Gi" # Modelle können viel RAM benötigen
                  cpu: "2"
              # Hinweis: Modelle werden in /root/.ollama gespeichert.
              # Ohne PersistentVolume gehen sie bei Pod-Neustart verloren.
    ```

2.  **`ollama.service.yaml`**:
    ```yaml
    # filepath: kubernetes/ollama.service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: ollama-backend # Dieser Name wird in der ConfigMap verwendet
      labels:
        app: ollama
    spec:
      type: ClusterIP # Nur intern im Cluster erreichbar
      selector:
        app: ollama # Selektiert Pods mit diesem Label
      ports:
        - port: 11434 # Service Port
          targetPort: 11434 # Container Port von Ollama
    ```

3.  **`webui.configmap.yaml`**:
    ```yaml
    # filepath: kubernetes/webui.configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: webui-config
    data:
      # Schlüssel entspricht dem erwarteten Environment-Variablen-Namen
      OLLAMA_API_BASE_URL: "http://ollama-backend:11434" # Verweist auf den Ollama Service
      # Zusätzliche optionale Konfigurationen für Open WebUI
      OLLAMA_BASE_URL: "http://ollama-backend:11434" # Oft synonym verwendet
      WEBUI_AUTH: "False" # Deaktiviert Authentifizierung für die Übung
      ENABLE_OLLAMA_API: "True" # Erlaubt der WebUI, die Ollama API direkt zu nutzen
    ```

4.  **`webui.deployment.yaml`**:
    ```yaml
    # filepath: kubernetes/webui.deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: webui-frontend
      labels:
        app: webui
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: webui
      template:
        metadata:
          labels:
            app: webui
        spec:
          containers:
            - name: open-webui
              image: ghcr.io/open-webui/open-webui:main # Öffentlich verfügbares Image
              ports:
                - containerPort: 8080
              env: # Umgebungsvariablen aus der ConfigMap injizieren
                - name: OLLAMA_API_BASE_URL
                  valueFrom:
                    configMapKeyRef:
                      name: webui-config # Name der ConfigMap
                      key: OLLAMA_API_BASE_URL # Schlüssel in der ConfigMap
                - name: OLLAMA_BASE_URL
                  valueFrom:
                    configMapKeyRef:
                      name: webui-config
                      key: OLLAMA_BASE_URL
                - name: WEBUI_AUTH
                  valueFrom:
                    configMapKeyRef:
                      name: webui-config
                      key: WEBUI_AUTH
                - name: ENABLE_OLLAMA_API
                  valueFrom:
                    configMapKeyRef:
                      name: webui-config
                      key: ENABLE_OLLAMA_API
              resources:
                requests:
                  memory: "1Gi" # Weniger anspruchsvoll als Ollama
                  cpu: "500m"
                limits:
                  memory: "2Gi"
                  cpu: "1"
    ```

5.  **`webui.service.yaml`**:
    ```yaml
    # filepath: kubernetes/webui.service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: webui-frontend # Dieser Name wird vom Ingress verwendet
      labels:
        app: webui
    spec:
      type: ClusterIP
      selector:
        app: webui
      ports:
        - port: 8080 # Service Port
          targetPort: 8080 # Container Port von WebUI
    ```

6.  **`ai-chat.ingress.yaml`**:
    ```yaml
    # filepath: kubernetes/ai-chat.ingress.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: ai-chat-ingress
      annotations:
        # NGINX Ingress spezifische Annotationen (optional, aber nützlich)
        nginx.ingress.kubernetes.io/rewrite-target: /
        nginx.ingress.kubernetes.io/proxy-read-timeout: "3600" # Längere Timeouts für Modell-Downloads
        nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    spec:
      ingressClassName: nginx # Wichtig: Name des Ingress Controllers
      rules:
      - host: chat.local # Hostname für den Zugriff
        http:
          paths:
          - path: / # Root-Pfad
            pathType: Prefix
            backend:
              service:
                name: webui-frontend # Verweist auf den WebUI Service
                port:
                  number: 8080 # Port des WebUI Service
    ```

### Ressourcen in Kubernetes deployen

Navigiere im Terminal zum Verzeichnis, das deine YAML-Dateien enthält (z.B. `kubernetes/`).

Wende alle YAML-Dateien an:
```bash
# Empfohlene Reihenfolge (obwohl `apply -f <dir>` dies oft handhabt):
kubectl apply -f kubernetes/webui.configmap.yaml
kubectl apply -f kubernetes/ollama.service.yaml
kubectl apply -f kubernetes/webui.service.yaml
kubectl apply -f kubernetes/ollama.deployment.yaml
kubectl apply -f kubernetes/webui.deployment.yaml
kubectl apply -f kubernetes/ai-chat.ingress.yaml

# Alternativ alle Dateien im Verzeichnis anwenden:
kubectl apply -f kubernetes/
```

### Deployment überprüfen und Zugriff testen

1.  **Status überprüfen**:
    ```bash
    kubectl get all # Zeigt alle Ressourcen im Default-Namespace
    kubectl get ingress
    # Warte, bis alle Pods den Status 'Running' haben (kann einige Minuten dauern)
    kubectl get pods -w
    ```
2.  **Logs prüfen**:
    ```bash
    kubectl logs -l app=ollama -f
    kubectl logs -l app=webui -f
    ```
3.  **Zugriff testen**:
    Öffne deinen Browser und rufe http://chat.local auf. Du solltest die Open WebUI Oberfläche sehen.
    *(Falls es nicht sofort funktioniert, gib dem Ingress Controller und den Services etwas Zeit, um sich zu initialisieren.)*

### Installation eines AI-Modells testen (Wichtiger Lernpunkt)

1.  Öffne die WebUI in deinem Browser (http://chat.local).
2.  Klicke auf das Zahnrad-Symbol (Einstellungen) oben rechts.
3.  Wähle den Reiter "Models".
4.  Unter "Pull a model from Ollama.ai" gib einen Modellnamen ein, z.B. `phi3:mini` oder `gemma2:2b`.
5.  Klicke auf den Download-Pfeil. Der Download kann einige Minuten dauern und erfordert die zuvor konfigurierten Ressourcen.
6.  Nach erfolgreichem Download erscheint das Modell in der Liste "Available Models". Wähle es aus.
7.  Gehe zurück zum Chat (Haus-Symbol) und teste den Chat mit dem Modell.

Wenn das Modell heruntergeladen wurde und der Chat funktioniert, hast du die Verbindung WebUI -> Ollama erfolgreich über Kubernetes Services und ConfigMap konfiguriert!

### Beobachtung zur Persistenz (Wichtiger Lernpunkt)

1.  **Modell herunterladen**: Stelle sicher, dass mindestens ein Modell über die WebUI heruntergeladen wurde und funktioniert.
2.  **Ollama Pod stoppen/neustarten**:
    ```bash
    # Option A: Pod manuell löschen (Deployment startet ihn neu)
    kubectl delete pod -l app=ollama

    # Option B: Deployment neu starten
    kubectl rollout restart deployment/ollama-backend
    ```
3.  **Neuen Pod abwarten**:
    ```bash
    kubectl get pods -l app=ollama -w
    ```
4.  **WebUI erneut prüfen**: Öffne die WebUI (http://chat.local), gehe zu Einstellungen -> Models.
    **Beobachtung**: Die zuvor heruntergeladenen Modelle sollten nun verschwunden sein!

**Erklärung**: Da der neue Ollama-Pod mit einem leeren Dateisystem startet und kein persistentes Volume für das Verzeichnis `/root/.ollama` gemountet wurde, gehen die Modelldaten verloren. Dies demonstriert die Limitation von `Deployments` für stateful Anwendungen ohne explizite Konfiguration von `PersistentVolumes` und `PersistentVolumeClaims`. Für eine produktive stateful Anwendung wäre ein `StatefulSet` die bessere Wahl.

## Deinstallation

```bash
# Alle erstellten Ressourcen entfernen (Reihenfolge umgekehrt zum Apply)
kubectl delete -f kubernetes/ai-chat.ingress.yaml
kubectl delete -f kubernetes/webui.deployment.yaml
kubectl delete -f kubernetes/ollama.deployment.yaml
kubectl delete -f kubernetes/webui.service.yaml
kubectl delete -f kubernetes/ollama.service.yaml
kubectl delete -f kubernetes/webui.configmap.yaml

# Alternativ alle mit `kubectl delete -f kubernetes/`

# Hosts-Eintrag entfernen (PowerShell als Administrator)
(Get-Content C:\Windows\System32\drivers\etc\hosts) | Where-Object {$_ -notmatch "chat.local"} | Set-Content C:\Windows\System32\drivers\etc\hosts
```

## Verfügbare AI-Modelle (Beispiele für `ollama pull <modell>`)

| Modell        | Größe  | Empfehlung                 |
|---------------|--------|----------------------------|
| `phi3:mini`   | ~2.3GB | Schnell, gut für Tests     |
| `gemma2:2b`   | ~1.6GB | Google's Gemma2, kompakt   |
| `llama3.1:8b` | ~4.7GB | Meta's Llama3.1, leistungsstark |
| `mistral:7b`  | ~4.1GB | Beliebtes Allround-Modell  |

*(Die genauen Größen können variieren.)*