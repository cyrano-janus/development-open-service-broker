# 🚀 OSB Broker Roadmap v2.1

> **Roadmap für den Go OSB Broker** — Zielbild: **Generischer Service Broker für
> Kubernetes-Operatoren in Enterprise-Umgebungen**
>
> Stand: 24. August 2026 · Ersetzt v1 (August 2026) und aktualisiert v2
> Referenz-Implementierung: [`osb-broker-go`](https://github.com/cyrano-janus/osb-broker-go)
> (Branch `feat/k8s-state-store`, OSB 2.17, bei Korifi verifiziert)

---

## ✅ Status: Phase 1 ABGESCHLOSSEN (Go, 24.08.2026)

Meilenstein **M1 „Hardened Reference"** ist für die Go-Referenz erreicht — alle
vier Phase-1-Features sind implementiert, getestet und im Cluster verifiziert:

| # | Feature | Status | Details |
|---|---------|--------|---------|
| 1.1 | **K8s-native Persistenz** | ✅ Erledigt (Go) | `StateStore`-Interface; `K8sStateStore` persistiert Instances/Bindings als JSON in ConfigMap `osb-broker-state`; Restart-E2E im Cluster bewiesen (Pod gekillt → Instanz + Binding überlebt) |
| 1.2 | **Basic Auth** | ✅ Erledigt (Go) | Middleware mit konstantzeitvergleichen Creds (`crypto/subtle`), 401 + `WWW-Authenticate`, `/healthz` ausgenommen, Creds via Secret (`BROKER_AUTH_USER/PASSWORD`) |
| 1.3 | **Structured Logging** | ✅ Erledigt (Go) | JSON-Logs pro Request mit UUID-Correlation-ID (`X-Correlation-ID`, inbound honoured), `X-OSB-Originating-Identity` im Logline für Audit |
| 1.4 | **Error Handling** | ✅ Erledigt (Go) | Zentrales `respondOSBError`: DELETE auf Nichtexistentes → 410 Gone (OSB-Spec), unknown service/plan → 400, Konflikte → 409 |

Zusätzlich erledigt und verifiziert: OSB-2.17-Vollzyklus, Registrierung bei
Korifi inkl. Marketplace-Sichtbarkeit und Service-Key-Lifecycle, `/healthz`
mit K8s-Probes, distroless-Dockerfile, RBAC least-privilege (Role scoped auf
die State-ConfigMap via `resourceNames`).

---

## 📊 Projektstatus je Implementierung

| Komponente | Repository | Stand |
|------------|-----------|-------|
| **Go Reference Broker** | [`osb-broker-go`](https://github.com/cyrano-janus/osb-broker-go) | 🟢 **Phase 1 komplett** (siehe oben), Detaildoku im Repo-README |
| **Java Reference Broker** | [`osb-broker-java`](https://github.com/cyrano-janus/osb-broker-java) | 🔴 **Offen** — noch keine Phase-1-Umsetzung (Persistenz, Auth, Logging, Error-Mapping stehen aus); Start empfohlen nach M2-Review der Go-Ergebnisse |
| **OSB Checker** | [`osb-checker`](./osb-checker) | Geplant als Conformance-Gate in CI (Phase 4.2) |

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

### Was blieb aus v1 und wie wurde es gelöst?

| v1-Phase | Urteil in v2 | Umsetzung Go |
|----------|--------------|--------------|
| Phase 1 Production Basics | ✅ übernommen — Persistenz K8s-nativ statt SQL | ✅ ConfigMap-StateStore |
| Phase 2 Second-Day Operations | ✅ übernommen — als Teil der Generic Engine | offen (Phase 2) |
| Phase 3 Advanced Features | ✅ größtenteils; „Async Queue" entfällt | offen (Phase 3) |
| Phase 4 Deployment & Ops | ⚡ teils vorgezogen | Dockerfile, Manifests, Probes, RBAC ✅ |
| Phase 5 Echte Services | ❌ ersetzt durch Definition Catalog | entfällt |

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

## 🟢 Phase 2: Generic Engine (Q1 2027) — NÄCHSTER SCHRITT

Ziel: YAML statt Code — der Kern des Generik.

| # | Feature | Umsetzung | Aufwand |
|---|---------|-----------|---------|
| 2.1 | **ServiceDefinition-API** | CRD oder ConfigMap-Format, Validierung beim Laden, Hot-Reload | Mittel |
| 2.2 | **Template-Renderer** | Go templates mit `.instanceID`, `.plan.*`, `.parameters`; strikte Escaping | Mittel |
| 2.3 | **CR-Lifecycle** | Provision = CR apply, Deprovision = CR delete, Update = CR patch (Plan-Wechsel!) | Mittel |
| 2.4 | **Readiness-Polling** | JSONPath auf CR-Status → LastOperation-Mapping inkl. Timeout/Failure-Erkennung | Einfach |
| 2.5 | **Binding-Extraktion** | Secret-Referenz rendern, Felder filtern/mappen, Rotation-fähig (Bind erneut = frisches Lesen) | Einfach |
| 2.6 | **RBAC-scoped ServiceAccount** | Erweiterung des bestehenden least-privilege-Role auf Space-Namespaces | Einfach |

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

| Meilenstein | Inhalt | Status / geplant |
|-------------|--------|------------------|
| **M1: Hardened Reference (Go)** | Phase 1 komplett | ✅ **Abgeschlossen 24.08.2026** |
| **M1a: Hardened Reference (Java)** | Phase 1 im Java-Broker nachziehen | 🔴 Offen — Start nach M2-Retrospektive empfohlen |
| **M2: Generic Engine** | Phase 2, erster echter CNPG-Service über YAML | Q1 2027 (nächster Schritt) |
| **M3: Catalog & Second-Day** | Phase 3, drei Operator-Beispiele | Q2 2027 |
| **M4: Enterprise Release** | Phase 4, Helm + CI mit Checker-Gate | Q3 2027 |

---

## 🤝 Contributing

Wie gehabt: Feature aussuchen → Issue → Implementieren → PR. Zusätzlich in
v2: **Jede ServiceDefinition ist ein eigener Beitrag** — ein Operator-YAML
mit Test reicht völlig, um das Angebot zu erweitern.

---

## 🌟 Vision

> *„Ein Plattform-Team legt eine YAML-Datei in Git — und ab sofort kann jeder
> Entwickler `cf create-service` für diesen Dienst nutzen. Keine Tickets,
> keine Broker-Entwicklung, volle Governance."*

**Lasst uns Cloud Foundry stark machen!** 💪
