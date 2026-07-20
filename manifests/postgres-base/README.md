# postgres-base

Herbruikbare Kustomize base voor een eigen Postgres-instance per app: een
StatefulSet (single replica, iSCSI-backed volume via democratic-csi), een
headless Service, en een ConfigMap voor de niet-geheime env-vars. Gebaseerd
op het patroon dat al gebruikt wordt in o.a. home-assistant en dsmr-reader.

Deze map wordt nooit zelf gedeployed (geen eigen ArgoCD Application, geen
`overlay/`) — hij bestaat puur om vanuit de `kustomization.yaml` van een
andere app te worden meegenomen via `resources:`.

## Wat de base wél en niet regelt

**Wel:** StatefulSet + headless Service + ConfigMap, met `POSTGRES_DB` en
`POSTGRES_USER` als placeholder-waarden (`app`/`app`), een `data`
volumeClaimTemplate van 10Gi op `freenas-iscsi-manual-csi`, en pg_isready
liveness/readiness probes.

**Niet:** de `PersistentVolume` (iSCSI IQN + grootte zijn per app uniek — die
hoort net als nu bij de app zelf), en het `Secret` met `POSTGRES_PASSWORD`
(SealedSecrets zijn per naam+namespace versleuteld, dus die maak je ook zelf
aan, met dezelfde naam als hieronder verwacht).

## Gebruik in een app

**Belangrijk:** de StatefulSet/Service in deze base delen allemaal het label
`app: postgresql-database`. Gebruik je deze base voor twee apps in dezelfde
namespace (zoals `tools`, waar meerdere apps samen draaien), dan matchen hun
selectors elkaar zonder een extra, per-app uniek label — de ene app zou dan
de pods van de andere als "eigen" kunnen zien. Voeg daarom **altijd** een
eigen `app-instance`-label toe via de `labels:` transformer, zoals hieronder.

In de `kustomization.yaml` van je app (naast je eigen `PersistentVolume` en
`SealedSecret` voor `POSTGRES_PASSWORD`):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../postgres-base/base
  - manifest.yaml # jouw eigen PV + SealedSecret (naam: postgresql-secret) + app zelf

namePrefix: mijnapp-

labels:
  - includeSelectors: true
    pairs:
      app-instance: mijnapp-postgresql

patches:
  - target:
      kind: ConfigMap
      name: postgresql-env
    patch: |-
      - op: replace
        path: /data/POSTGRES_DB
        value: mijnapp
      - op: replace
        path: /data/POSTGRES_USER
        value: mijnapp
  - target:
      kind: StatefulSet
      name: postgresql
    patch: |-
      - op: replace
        path: /spec/volumeClaimTemplates/0/spec/resources/requests/storage
        value: 20Gi
```

Je eigen `SealedSecret` moet **exact** `postgresql-secret` heten (vóór de
`namePrefix`) met key `POSTGRES_PASSWORD` — kustomize hernoemt 'm dan
automatisch mee naar `mijnapp-postgresql-secret` en zet de `secretKeyRef` in
de StatefulSet daar ook automatisch naar toe. Zelfde principe voor de
`serviceName`/headless Service: die hoef je nergens los te patchen, dat lost
`namePrefix` vanzelf op.

De app zelf verbindt met `mijnapp-postgresql-hl:5432` (of
`mijnapp-postgresql-hl.<namespace>.svc.cluster.local` van buiten de
namespace).

### Wil je een andere postgres-versie of extension (zoals immich's pgvector-image)?

Patch de image direct:

```yaml
  - target:
      kind: StatefulSet
      name: postgresql
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: docker.io/postgres:17-alpine
```

## Bestaande apps migreren?

Dit is bewust niet gedaan voor home-assistant, dsmr-reader of
open-family-finance — hun StatefulSets draaien al met bestaande data.
Migreren kan, maar is een aparte, voorzichtige actie (PVC-namen/labels
veranderen bij een StatefulSet raakt al snel de bestaande PV-binding) en
geen automatisch gevolg van het toevoegen van deze base.
