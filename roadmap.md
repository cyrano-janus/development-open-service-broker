# 🚀 OSB Broker Roadmap v2.4

> **Roadmap für den Go OSB Broker** — Zielbild: **Generischer Service Broker für
> Kubernetes-Operatoren in Enterprise-Umgebungen**
>
> Stand: 25. August 2026 · Ersetzt v1–v2.3 · **Phase 1 + 2 + 3 + 4.2 (Go) ABGESCHLOSSEN**
> Referenz-Implementierung: [`osb-broker-go`](https://github.com/cyrano-janus/osb-broker-go)
> (`main @ a14bed1`, OSB 2.17, CI mit Conformance-Gate grün)

---

## ✅ Status: Phase 1 UND Phase 2 ABGESCHLOSSEN (Go, 24.08.2026)

Meilensteine **M1 „Hardened Reference"** und **M2 „Generic Engine"** sind für
die Go-Referenz erreicht — beide im Cluster verifiziert:

| # | Feature | Status | Details |
|---|---------|--------|---------|
| 1.1 | **K8s-native Persistenz** | ✅ | `StateStore`-Interface; `K8sStateStore` persistiert in ConfigMap `osb-broker-state`; Restart-E2E bewiesen |
| 1.2 | **Basic Auth** | ✅ | Konstantzeitvergleich, Secret-Injection, `/healthz` ausgenommen, disabled-Modus mit Warnung |
| 1.3 | **Structured Logging** | ✅ | JSON-Logs, UUID-Correlation-ID (`X-Correlation-ID`), Audit via `X-OSB-Originating-Identity` |
| 1.4 | **Error Handling** | ✅ | Zentrales Mapping: DELETE fehlend → 410 Gone, unbekanntes service/plan → 400, Konflikt → 409 |
| 2.1 | **ServiceDefinition-API** | ✅ | YAML-Parse/Validate (`broker.osb.io/v1alpha1`), Pflichtfelder, Duplicate-Plan-Ablehnung, LoadFromDir-Katalog |
| 2.2 | **Template-Renderer** | ✅ | Go-Templates mit Dual-Style-Datenzugriff (`.instanceID`/`.InstanceID`), `missingkey=error` |
| 2.3 | **CR-Lifecycle** | ✅ | ApplyCR/Create-or-Update, DeleteCR, GetCR/GetCRStatus über controller-runtime Client |
| 2.4 | **Readiness-Polling** | ✅ | gjson auf CR-Status (`status.conditions.#(type=="Ready").status`) → last_operation-Mapping |
| 2.5 | **Binding-Extraktion** | ✅ | Operator-Secret `<id>-app` lesen, Key-Filter, Rotation-fähig |
| 2.6 | **RBAC-scoped SA** | ✅ | Least-privilege Role auf State-ConfigMap; ClusterRole nur für operator-spezifische CRs |

**E2E-Beweis M2:** `cf create-service cnpg-postgresql large my-real-pg`
erzeugte einen echten CloudNativePG-Cluster (3 Instanzen, 10Gi),
`psql -c "SELECT 'E2E OK'"` erfolgreich im Pod, echte Credentials aus dem
Operator-Secret `<id>-app` als Service-Key ausgeliefert.

---

## 📊 Projektstatus je Implementierung

| Komponente | Repository | Stand |
|------------|-----------|-------|
| **Go Reference Broker** | [`osb-broker-go`](https://github.com/cyrano-janus/osb-broker-go) | 🟢 **Phase 1 + 2 komplett** (auf `main`, 84 Tests grün); Detaildoku im Repo-README |
| **Java Reference Broker** | [`osb-broker-java`](https://github.com/cyrano-janus/osb-broker-java) | 🔴 **Offen** — Phase 1 (Persistenz, Auth, Logging, Error-Mapping) und Phase 2 (Generic Engine) stehen noch aus; Start empfohlen nach M3-Retrospektive der Go-Ergebnisse |
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
| Phase 2 Second-Day Operations | ✅ übernommen — als Teil der Generic Engine | ✅ Engine implementiert |
| Phase 3 Advanced Features | ✅ größtenteils; „Async Queue" entfällt | offen (Phase 3) |
| Phase 4 Deployment & Ops | ⚡ teils vorgezogen | Dockerfile, Manifests, Probes, RBAC ✅ |
| Phase 5 Echte Services | ❌ ersetzt durch Definition Catalog | ✅ erster Beweis: CNPG |

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
      - id: plan-small-0000-0000-000000000001
        name: small
        params:
          storageSize: 1Gi
          instances: 1
      - id: plan-large-0000-0000-000000000002
        name: large
        params:
          storageSize: 10Gi
          instances: 3
  provision:
    apiVersion: postgresql.cnpg.io/v1
    kind: Cluster
    template: |
      apiVersion: postgresql.cnpg.io/v1
      kind: Cluster
      metadata:
        name: {{ .instanceID }}
      spec:
        instances: {{ .plan.instances }}
        storage:
          size: {{ .plan.storageSize }}
  readiness:
    statusJSONPath: 'status.conditions.#(type=="Ready").status'
    expectedValue: "True"
    timeoutSeconds: 600
  bind:
    credentialsFromSecret: "{{ .instanceID }}-app"
```

Der Broker führt daraus aus:

| OSB-Call | Aktion |
|----------|--------|
| `GET /v2/catalog` | Alle ServiceDefinitions → Katalog |
| `PUT /service_instances` | Template rendern (Plan-Params) → CR anlegen |
| `GET .../last_operation` | CR-Status via gjson pollen → `in progress/succeeded/failed` |
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

**Dieser Ablauf ist live bewiesen** — siehe Status oben.

---

## 🟢 Phase 3: Definition Catalog & Second-Day — ✅ ABGESCHLOSSEN (24.08.2026)

Ersatz für v1-Phase 5 — dieselben Dienste, aber als YAML:

| # | Feature | Status | Details |
|---|---------|--------|---------|
| 3.1 | **Catalog-Repo** | ✅ | `definitions/` (7 Stück): CNPG, Redis, MinIO, Valkey, RabbitMQ, Redpanda, SeaweedFS; Catalog-Guard-Test verhindert Definition-Drift. **E2E über Korifi (25.08.):** RabbitMQ ✅ und SeaweedFS ✅ (create succeeded, Ready=True, Service-Key mit echten Credentials). Valkey 🔴 blockiert (hyperspike-Operator braucht bitnami/bash-Images), Redpanda 🔴 blockiert (Operator 26.x = Helm-Wrapper ohne Spec-Felder → braucht 4.6) |
| 3.2 | **Update-Logik** | ✅ | Plan-Wechsel → Template re-render → ApplyCR; **No-op-Erkennung** (identisches Spec+Labels = kein Apply, kein resourceVersion-Bump); `allowedParameters`-Whitelist pro Plan, unbekannte Parameter → 400 |
| 3.3 | **Credential Rotation** | ✅ | Integrationstest: Rebind liest rotiertes Secret frisch (kein Caching); Key-Filter pro Definition |
| 3.4 | **Instance Sharing** | ⏭️ Vertagt | OSB-Feature „shared" — Bedarf im Enterprise-Kontext gering, Aufwand hoch; Re-Evaluierung mit Phase 4 |

**E2E-Beweis M3 (direkt gegen den Broker):** `PATCH` small→large auf die
live-provisionierte CNPG-Instanz skalierte den Cluster **1 → 3 Instanzen**,
beide Replicas streaming, `psql` erfolgreich. Reverse-Richtung liefert
sauberen Fehler (CNPG verhindert PVC-Schrumpfung — korrektes Operator-
Verhalten). Zusätzlich zwei Produktions-Bugs gefunden und gefixt:
CNPG-Webhook lehnt bare-GUID-Namen ab (`safeName` mit immer `osb-`-Präfix)
und Kind-StorageClass ohne Volume-Expansion blockierte Reconciles.

---

## ⚠️ Bekannte Plattform-Limitation: Korifi v0.18.0 (OSB-Update-Pfad)

Bei der Verifikation des Update-Flows über Cloud Foundry wurde eine
**Korifi-seitige Lücke** identifiziert (kein Broker-Fehler):

| Aspekt | Beobachtung |
|--------|-------------|
| Symptom | `cf update-service -p <plan>` zeigt „OK", aber der Broker erhält **niemals** den OSB-`PATCH`-Call |
| Ursache | Der CFServiceInstance-Controller v0.18.0 scheitert beim Lesen des Parameters-Secrets (`parameters: null` statt `{}`) und beendet den Reconcile, bevor der OSB-Update-Call ausgeführt wird |
| Isolation | Direkter OSB-PATCH gegen denselben Broker/dieselbe Instanz funktioniert vollständig (siehe M3-Beweis); Provision/Bind/LastOperation laufen auch über Korifi einwandfrei |
| Workaround | Updates direkt per OSB-PATCH auslösen (Skripte/Terraform) oder auf Korifi ≥ neuere Version upgraden |
| Status | Dokumentiert als Plattform-Limitation; Broker-seitig nichts zu tun. Re-Test bei nächstem Korifi-Upgrade |

---

## 🟣 Phase 4: Enterprise Operations (Q3 2027)

| # | Feature | Status | Details |
|---|---------|--------|---------|
| 4.1 | **Helm Chart** | ✅ **Abgeschlossen 25.08.2026** | Chart unter `deploy/helm/`: Deployment/Service/RBAC (namespaced State-Role + ClusterRole nur für Operator-CRs), Auth-Secret (create oder existingSecret), Definitions-ConfigMap aus Values. Verifiziert mit `helm lint` + 16 Render-Checks über drei Szenarien (Default ohne Definitions-Dateien, kind-Values mit 4 gebündelten Definitionen, existingSecret-Pfad). Bug gefunden & gefixt: Mount auf nicht existierende ConfigMap bei `create=true` ohne `files` |
| 4.2 | **CI/CD + Conformance-Gate** | ✅ **Abgeschlossen 25.08.2026** | Drei-Level GitHub Actions: L1 vet/test-race/build (~30s) → L2 Conformance-Gate: ephemeral kind-Cluster + CNPG + Broker aus dem PR, geprüft vom eigenen `osb-checker`-Tool (Auth, Catalog-Shape, Lifecycle inkl. 410-Gone, Error-Mapping; ~3 min) → L3 Release bei Tags (Multi-Arch nach GHCR + GitHub Release). Free-Tier-tauglich (Public Repo gratis). Der Gate hat sofort gelohnt: `.gitignore`-Bug entdeckt (internal/broker war nie committed) und 410-Gone-Lücke im Definition-Deprovision gefunden |
| 4.3 | **Metrics** | 🔴 Offen | Prometheus: Request-Latenzen, Provision-Dauer, aktive Instanzen, Fehlerquoten |
| 4.4 | **OpenAPI-Doku** | ✅ **Abgeschlossen 25.08.2026** | Vollständige OpenAPI-3.0.3-Spec (alle 11 Router-Operationen, deckungsgleich mit dem Gin-Router verifiziert) + ServiceDefinition-JSON-Schema (Draft-07, gegen alle shipped Definitions geprüft). Selbst-Hosted: `GET /openapi.yaml` und `GET /schemas/service-definition.schema.json`, unauthentifiziert, zur Compile-Zeit via go:embed eingebettet — das Binary dokumentiert sich selbst. 3 Handler-Tests schützen die Doku-Routen |
| 4.5 | **mTLS/OAuth2 (optional)** | ⏭️ Vertagt | Für Umgebungen ohne NetworkPolicy-Isolation; nur bei konkretem Bedarf |
| **4.6** | **Engine: Multi-Doc-Templates** | 🔴 Offen (nächste Engine-Erweiterung) | `provision.template` auf mehrere durch `---` getrennte Manifeste erweitern — schaltet Strimzi-Kafka (KafkaNodePool + Kafka), RocketMQ und Dgraph als Angebote frei. Details: README „Nächste Schritte" im Broker-Repo. Voraussetzung für die nächsten Catalog-Erweiterungen |

---

## 🔵 Java-Nachzug (parallel möglich)

Die Generic-Engine-Architektur ist sprachneutral. Der Java-Broker hinkt
bewusst hinterher, damit die Go-Seite die Design-Fehler zuerst findet:

| Feature | Beschreibung |
|---------|--------------|
| Phase 1-Java | Persistenz (JPA oder K8s-nativ analog ConfigMap), Basic Auth, strukturierte Logs, Error-Mapping |
| Phase 2-Java | ServiceDefinition-Parser (Jackson/YAML), Template-Renderer, CR-Lifecycle über Java-Client |

Empfehlung: Start nach Abschluss von Phase 3 (Go), dann Portierung mit den
Erkenntnissen aus Catalog + Update-Praxis.

---

## 📅 Meilensteine

| Meilenstein | Inhalt | Status / geplant |
|-------------|--------|------------------|
| **M1: Hardened Reference (Go)** | Phase 1 komplett | ✅ **Abgeschlossen 24.08.2026** |
| **M2: Generic Engine (Go)** | Phase 2 komplett, CNPG E2E bewiesen | ✅ **Abgeschlossen 24.08.2026** |
| **M3: Catalog & Second-Day** | Phase 3, Update-E2E direkt gegen Broker bewiesen | ✅ **Abgeschlossen 24.08.2026** |
| **M4a: CI/CD + Conformance-Gate** | Phase 4.2, Pipeline grün auf main | ✅ **Abgeschlossen 25.08.2026** |
| **M1a/M2a: Java-Nachzug** | Phase 1+2 im Java-Broker (siehe „Java-Nachzug") | 🔴 Offen — jetzt startklar (Go-Design stabil) |
| **M4b: Enterprise Release** | Helm Chart (✅), OpenAPI-Doku (✅), Metrics | 🔴 Teils offen — 4.3 Metrics + 4.6 Multi-Doc-Templates |

> **Korifi-Hinweis:** v0.18.0 leitet OSB-Update-PATCHes nicht weiter —
> siehe „Bekannte Plattform-Limitation" oben.

---

## 🤝 Contributing

Wie gehabt: Feature aussuchen → Issue → Implementieren → PR. Zusätzlich in
v2: **Jede ServiceDefinition ist ein eigener Beitrag** — ein Operator-YAML
mit Test reicht völlig, um das Angebot zu erweitern.

---

## 🌟 Vision — und wie nah wir ihr sind

> *„Ein Plattform-Team legt eine YAML-Datei in Git — und ab sofort kann jeder
> Entwickler `cf create-service` für diesen Dienst nutzen. Keine Tickets,
> keine Broker-Entwicklung, volle Governance."*

Genau dieser Ablauf läuft heute real: Die CNPG-Definition liegt im Broker,
`cf create-service cnpg-postgresql large my-db` provisioniert einen echten
PostgreSQL-Cluster. **Was in v1 Vision war, ist jetzt ein funktionierender
Verbesserungszyklus.** 💪

**Lasst uns Cloud Foundry stark machen!**
