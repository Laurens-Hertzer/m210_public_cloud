# Prüfungs-Spickzettel: 3-Tier Cloud App auf OpenShift
> Teil 2 – Theoriefragen & Debugging

---

## 1. ARCHITEKTUR: Was ist eine 3-Tier-App?

```
[ Browser / Client ]
        │  HTTPS
        ▼
[ Tier 1: Frontend ]     → nginx, React, Vue, Angular
        │  HTTP intern
        ▼
[ Tier 2: Backend/API ]  → Node.js, Python, Java, Go
        │  DB-Protokoll
        ▼
[ Tier 3: Datenbank ]    → PostgreSQL (5432), MariaDB (3306)
```

Vorteile der Trennung: Skalierbarkeit, Sicherheit, Wartbarkeit

---

## 2. KUBERNETES / OPENSHIFT – Konzepte

| Objekt | Zweck |
|---|---|
| **Pod** | Kleinste Einheit, läuft Container |
| **Deployment** | Verwaltet Pods, Rollouts, Replicas |
| **Service** | Stabiler interner DNS-Name + LoadBalancing |
| **Route** | OpenShift-spezifisch: externer HTTPS-Zugang |
| **PVC** | Persistent Volume Claim – Datenspeicher für DB |
| **Secret** | Verschlüsselte Konfigurationswerte (base64) |
| **HPA** | Horizontal Pod Autoscaler – auto-skaliert Pods |
| **ConfigMap** | Unkritische Konfigurationswerte |
| **Namespace** | Logische Trennung von Ressourcen |

### Wie kommunizieren Pods intern?
→ Über den **Service-Namen** (DNS). Beispiel:
  `my-app-database` → löst intern zu ClusterIP auf.
  Der DB_HOST im Secret muss = Service-Name sein!

---

## 3. BASE64 – Secrets encodieren/decodieren

```bash
# Enkodieren (für Secret data:)
echo -n "meinpasswort" | base64
# Dekodieren (zum Prüfen)
echo "bWVpbnBhc3N3b3J0" | base64 -d
```
> **Wichtig:** `echo -n` verhindert Newline am Ende!

---

## 4. WICHTIGE oc / kubectl BEFEHLE

```bash
# Projekt / Namespace
oc new-project mein-projekt
oc project mein-projekt

# Ressourcen deployen
oc apply -f 010-PVC.yaml
oc apply -f .              # alle YAMLs im Ordner

# Status prüfen
oc get pods
oc get pods -w             # live watch
oc get all                 # alles auf einmal
oc get pvc
oc get secret

# Logs anzeigen
oc logs <pod-name>
oc logs <pod-name> -f      # follow/live
oc logs <pod-name> --previous  # letzter abgestürzter Pod

# Pod beschreiben (Events & Fehler!)
oc describe pod <pod-name>
oc describe deployment <name>

# In Pod einloggen
oc exec -it <pod-name> -- /bin/bash
oc exec -it <pod-name> -- /bin/sh   # falls kein bash

# Ressource löschen
oc delete pod <pod-name>   # Pod neu starten (Deployment erstellt neuen)
oc delete -f .             # alle YAMLs rückgängig
```

---

## 5. DEBUGGING – Typische Fehler & Lösungen

### Pod startet nicht / CrashLoopBackOff
```bash
oc describe pod <name>   # → Events lesen!
oc logs <name>           # → App-Fehler lesen
oc logs <name> --previous
```
**Häufige Ursachen:**
- Falsches Image oder Image nicht erreichbar → `ImagePullBackOff`
- Falscher Port im Deployment
- Secret-Key falsch geschrieben
- DB noch nicht bereit (race condition) → `depends_on` / Retry in App

### ImagePullBackOff
```bash
oc describe pod <name>   # → "Failed to pull image..."
```
- Image-Name/Tag falsch?
- Fehlende ImagePullSecrets für private Registry?
- Netzwerkproblem zur Registry?

### DB-Verbindung schlägt fehl
Checkliste:
1. Service-Name korrekt? (`oc get svc`)
2. Port korrekt? (PostgreSQL=5432, MariaDB=3306)
3. Secret-Werte korrekt? (`oc get secret <name> -o yaml`)
4. DB-Pod läuft? (`oc get pods`)
5. Env-Variablen im App-Pod prüfen:
   ```bash
   oc exec -it <app-pod> -- env | grep DB
   ```

### Secret-Werte prüfen
```bash
oc get secret my-app-secret -o jsonpath='{.data.database-password}' | base64 -d
```

### PVC bleibt "Pending"
- StorageClass existiert? (`oc get storageclass`)
- Genug Speicher verfügbar?
- AccessMode kompatibel? (`ReadWriteOnce` = nur 1 Node)

### Route nicht erreichbar
```bash
oc get route          # → URL anzeigen
oc describe route     # → Fehler prüfen
```
- TLS-Zertifikat Problem?
- Service-Name in Route falsch?
- App-Pod läuft?

---

## 6. PORTS – Schnellreferenz

| Service | Standard-Port |
|---|---|
| HTTP | 80 |
| HTTPS | 443 |
| PostgreSQL | 5432 |
| MariaDB / MySQL | 3306 |
| MongoDB | 27017 |
| Redis | 6379 |
| Node.js (typisch) | 3000 |
| Python Flask | 5000 |
| Java Spring | 8080 |
| nginx | 80 / 443 |

---

## 7. DOCKER – Wichtige Befehle

```bash
# Image bauen
docker build -t my-app:v1 .
docker build -t ghcr.io/user/my-app:v1 .

# Image pushen
docker push ghcr.io/user/my-app:v1

# Container starten
docker run -d -p 8080:8080 my-app:v1

# Docker Compose
docker compose up -d          # starten
docker compose down           # stoppen
docker compose logs -f        # logs
docker compose ps             # status

# In Container einloggen
docker exec -it <container> /bin/bash

# Laufende Container
docker ps
docker ps -a                  # auch gestoppte
```

---

## 8. HEALTH CHECKS & READINESS

### Liveness vs Readiness Probe
- **Liveness**: Ist der Container noch am Leben? (→ Neustart wenn nein)
- **Readiness**: Ist der Container bereit für Traffic? (→ Service schickt Traffic erst wenn ready)

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

## 9. HPA – Autoscaling erklärt

- HPA überwacht CPU/RAM der Pods
- Bei Überschreitung von `averageUtilization` → neue Pods starten
- Pods werden zwischen `minReplicas` und `maxReplicas` skaliert
- Wichtig: `resources.requests` im Deployment muss gesetzt sein!

```bash
oc get hpa          # Status anzeigen
oc describe hpa     # Details
```

---

## 10. OPENSHIFT vs KUBERNETES Unterschiede

| Feature | Kubernetes | OpenShift |
|---|---|---|
| Externer Zugang | Ingress | **Route** |
| Registry | extern | integriert |
| Security | weniger restriktiv | **SCC** (strenger) |
| Web-UI | Dashboard (optional) | integriert |
| CLI | kubectl | **oc** (= kubectl + mehr) |

> OpenShift läuft standardmässig als nicht-root! Falls Container root braucht → SCC anpassen.

---

## 11. CHECKLIST – Deployment in der Prüfung

```
[ ] 1. Projekt/Namespace erstellen
[ ] 2. Alle MY-APP Platzhalter ersetzen
[ ] 3. Secret-Werte base64-encodieren
[ ] 4. DB-Host im Secret = Service-Name der DB
[ ] 5. Ports konsistent (Deployment ↔ Service ↔ Route)
[ ] 6. PVC Storage-Name vorhanden
[ ] 7. In Reihenfolge deployen: PVC → Secret → Deployment → Service → Route → HPA
[ ] 8. oc get pods → alle Running?
[ ] 9. oc logs <app-pod> → keine Fehler?
[ ] 10. Route aufrufen → App erreichbar?
```

---

## 12. VOLUMES – Warum braucht die DB ein PVC?

Container sind **ephemeral** (flüchtig) – ohne Volume gehen alle Daten beim Neustart verloren.
PVC = Persistent Volume Claim → reserviert echten Speicher auf dem Cluster.

Mount-Pfade:
- PostgreSQL: `/var/lib/pgsql/data`
- MariaDB: `/var/lib/mysql/data`
- MySQL: `/var/lib/mysql`
