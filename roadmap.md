# 🚀 OSB Broker Roadmap v2

> **Roadmap für Go OSB Broker** — Zielbild: **Generischer Service Broker für
> Kubernetes-Operatoren in Enterprise-Umgebungen**
>
> Stand: August 2026 · Ersetzt Roadmap v1 (vom August 2026)
> Basis: [`osb-broker-go`](https://github.com/cyrano-janus/osb-broker-go) (OSB 2.17, bei Korifi verifiziert)

---

## 🎯 Strategische Kurskorrektur (v1 → v2)

Roadmap v1 sah in Phase 5 **sechs einzelne service-spezifische Broker** vor
(PostgreSQL-, Redis-, RabbitMQ-, MongoDB-Broker …). Das widerspricht dem
eigentlichen Ziel: Jeder Broker wäre eine neue Codebasis mit eigenen
Abhängigkeiten, eigenem Betrieb, eigenem Security-Review — der klassische
Broker-Wildwuchs.

**v2 dreht das um:**

```
v1:  Ein Broker pro Service-Typ        (N Codebasen, N Deployments)
v2:  Eine Broker-Engine + N YAML-Definitionen   (1 Codebase, Config statt Code)
```

Ein einziger, gehärteter Broker-Prozess rendert deklarative
**ServiceDefinitions** auf Kubernetes-CustomResources beliebiger Operatoren.
Neuer Service im Marketplace = eine YAML-Datei, kein Code, kein Deployment.

### Was bleibt aus v1?

| v1-Phase | Urteil in v2 |
|----------|--------------|
| Phase 1 Production Basics | ✅ übernommen — Persistenz aber K8s-nativ statt SQL |
| Phase 2 Second-Day Operations | ✅ übernommen — als Teil der Generic Engine |
| Phase 3 Advanced Features | ✅ größtenteils übernommen; „Async Queue" entfällt (Operatoren sind selbst asynchron; der Broker pollt CR-Status) |
| Phase 4 Deployment & Ops | ⚡ teils erledigt (Dockerfile, K8s-Manifests, Probes); Rest: Helm, CI |
| Phase 5 Echte Services | ❌ ersetzt durch **Definition Catalog** (YAML statt Code) |

---

## 🧩 Kernstück: ServiceDefinition (deklarativ, kein Code)

Jedes Marketplace-Angebot ist ein YAML-Dokument:

```yaml
apiVersion: broker.osb.io/v1alpha1
kind: ServiceDefinition
metadata:
  name: cnpg-postgresql
spec:
  offering:
    id: f48a9e21-cnpg-0000-0000-000000000001
    name: cnpg-postgresql
    description: "CloudNativePG PostgreSQL clusters"
    tags: [postgresql, database]
    plans:
      - name: small
        description: "Single instance, 1Gi"
        params:
          storageSize: 1Gi
          instances: 1
      - name: large
        description: "HA, 3 instances, 10Gi"
        params:
          storageSize: 10Gi
          instances: 3

  provision:
    apiVersion: postgresql.cnpg.io/v1
    kind: Cluster
    namespaceFrom: space          # cf-space-guid -> k8s namespace
    template: |
      metadata:
        name: {{ .instanceID }}
      spec:
        instances: {{ .plan.instances }}
        storage:
          size: {{ .plan.storageSize }}

  readiness:
    statusJSONPath: '.conditions[?(@.type=="Ready")].status'
    expectedValue: "True"
    timeoutSeconds: 600

  bind:
    credentialsFromSecret: "{{ .instanceID }}-app"   # CNPG-Konvention
```

Der Broker führt daraus aus:

| OSB-Call | Aktion |
|----------|--------|
| `GET /v2/catalog` | Alle ServiceDefinitions → Katalog |
| `PUT /service_instances` | Template rendern (Plan-Params) → CR anlegen |
| `GET .../last_operation` | CR-Status via JSONPath pollen → `in progress/succeeded/failed` |
| `PUT .../bindings` | Secret `<instance>-app` lesen → Credentials zurückgeben |
| `DELETE ...` | CR löschen / Binding entfernen |

**Funktioniert mit jedem Operator**, der (a) CRDs und (b) ein
Credentials-Secret erzeugt — der De-facto-Standard moderner Operatoren.

```text
cf create-service cnpg-postgresql large mydb
     │
     ▼
Broker Engine ──► rendert CR-Template ──► CR im Space-Namespace
     │                                          │
     │                                    Operator reconciliert
     │                                    (Pods, Services, Secrets)
     │                                          │
     ├─ last_operation ◄─── pollt CR-Status ◄───┘
     │
     ▼ bind: liest <id>-app-Secret
VCAP_SERVICES in der App
```

---

## 🔵 Phase 1: Production Basics (Q4 2026)

Ziel: Der Broker selbst ist betriebssicher — **ohne Abhängigkeit von einem
externen Service-Deployment** (bewusst kein SQL/Redis als Broker-Store).

| # | Feature | Umsetzung | Aufwand |
|---|---------|-----------|---------|
| 1.1 | **K8s-native Persistenz** | Instances/Bindings als eigene CRDs (`OSBInstance`, `OSBBinding`) oder annotierte Objekte im Broker-Namespace. Pod-Restart-safe, keine externe DB | Mittel |
| 1.2 | **Basic Auth** | Credentials aus Kubernetes-Secret; 401 + WWW-Authenticate; Brute-force-Delay. Vorbereitet für OAuth2/OIDC später | Einfach |
| 1.3 | **Structured Logging** | JSON-Logs, Correlation-ID pro Request, Auswertung von `X-OSB-Originating-Identity` (Audit: *wer* hat *was* angelegt) | Einfach |
| 1.4 | **Error Handling** | Zentrale OSB-Fehler-Mapping-Schicht (400/409/410/422 korrekt je Fall), einheitliche Response-Struktur | Einfach |

Akzeptanzkriterium: Pod-Restart verliert keine Instance/Binding; alle
Endpoints verhalten sich spec-konform ohne Header-Basteln.

---

## 🟢 Phase 2: Generic Engine (Q1 2027)

Ziel: YAML statt Code — der Kern des Generik.

| # | Feature | Umsetzung | Aufwand |
|---|---------|-----------|---------|
| 2.1 | **ServiceDefinition-API** | CRD oder ConfigMap-Format, Validierung beim Laden, Hot-Reload | Mittel |
| 2.2 | **Template-Renderer** | Go templates mit `.instanceID`, `.plan.*`, `.parameters`; strikte Escaping | Mittel |
| 2.3 | **CR-Lifecycle** | Provision = CR apply, Deprovision = CR delete, Update = CR patch (Plan-Wechsel!) | Mittel |
| 2.4 | **Readiness-Polling** | JSONPath auf CR-Status → LastOperation-Mapping inkl. Timeout/Failure-Erkennung | Einfach |
| 2.5 | **Binding-Extraktion** | Secret-Referenz rendern, Felder filtern/mappen, Rotation-fähig (Bind erneut = frisches Lesen) | Einfach |
| 2.6 | **RBAC-scoped ServiceAccount** | Broker darf nur in Space-Namespaces schreiben (Least Privilege, Enterprise-Pflicht) | Einfach |

Erster End-to-End: **CNPG-Definition** gegen Korifi (`cf create-service`
erzeugt echten PostgreSQL-Cluster).

---

## 🟡 Phase 3: Definition Catalog & Second-Day (Q2 2027)

Ersatz für v1-Phase 5 — dieselben Dienste, aber als YAML:

| # | Feature | Beschreibung |
|---|---------|--------------|
| 3.1 | **Catalog-Repo** | `definitions/` mit Beispielen: CNPG (PostgreSQL), Redis-Operator, MinIO |
| 3.2 | **Update-Logik final** | Plan-Wechsel → CR-Patch verifizieren; Parameter-Validierung per JSON-Schema pro Plan |
| 3.3 | **Credential Rotation** | Unbind/Rebind liest Secrets frisch; optional Auto-Rotation-Annotation |
| 3.4 | **Instance Sharing (optional)** | OSB-Feature „shared": Bindings über Space-Grenzen, RBAC-geprüft |

---

## 🟣 Phase 4: Enterprise Operations (Q3 2027)

| # | Feature | Beschreibung | Aufwand |
|---|---------|--------------|---------|
| 4.1 | **Helm Chart** | Values: Definitions-Quelle, Auth-Secret, RBAC, Namespace-Selektoren | Mittel |
| 4.2 | **CI/CD** | GitHub Actions: vet/test/build/image; **Conformance-Gate: eigener osb-checker** läuft gegen jeden PR-Build | Mittel |
| 4.3 | **Metrics** | Prometheus: Request-Latenzen, Provision-Dauer, aktive Instanzen, Fehlerquoten | Mittel |
| 4.4 | **OpenAPI-Doku** | Broker-Endpoints + ServiceDefinition-Schema dokumentiert | Einfach |
| 4.5 | **mTLS/OAuth2 (optional)** | Für Umgebungen ohne NetworkPolicy-Isolation | Hoch |

---

## 📅 Meilensteine

| Meilenstein | Inhalt | Geplant |
|-------------|--------|---------|
| **M1: Hardened Reference** | Phase 1 komplett (persistenter, authentifizierter, beobachtbarer Broker) | Q4 2026 |
| **M2: Generic Engine** | Phase 2, erster echter CNPG-Service über YAML | Q1 2027 |
| **M3: Catalog & Second-Day** | Phase 3, drei Operator-Beispiele | Q2 2027 |
| **M4: Enterprise Release** | Phase 4, Helm + CI mit Checker-Gate | Q3 2027 |

---

## ✅ Bereits erledigt (aus v1/v2, Stand August 2026)

- ✅ OSB-2.17-Lebenszyklus vollständig (catalog/provision/bind/lastop/unbind/deprovision)
- ✅ Bei Korifi registriert und verifiziert (Marketplace, create-service, Service-Keys)
- ✅ `/healthz`-Endpoint + K8s-Probes
- ✅ Dockerfile (distroless, nonroot), Deployment-/Service-Manifests
- ✅ go vet clean, Test-Suite grün
- ✅ Binding-Schutz (Deprovision mit offenen Bindings → 410)

---

## 🤝 Contributing

Wie gehabt: Feature aussuchen → Issue → Implementieren → PR. Zusätzlich in
v2: **Jede ServiceDefinition ist ein eigener Beitrag** — ein Operator-YAML
mit Test reicht völlig, um das Angebot zu erweitern.

---

## 🌟 Vision

> *"Ein Plattform-Team legt eine YAML-Datei in Git — und ab sofort kann jeder
> Entwickler `cf create-service` für diesen Dienst nutzen. Keine Tickets,
> keine Broker-Entwicklung, volle Governance."*

**Lasst uns Cloud Foundry stark machen!** 💪
