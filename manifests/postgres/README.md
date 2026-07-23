# postgres-base

Herbruikbare Kustomize base voor een eigen Postgres-instance per app: PV +
StatefulSet (single replica, iSCSI-backed volume via democratic-csi),
headless Service, ConfigMap voor de niet-geheime env-vars, en een
placeholder-SealedSecret voor het wachtwoord. Gebaseerd op het patroon dat
al gebruikt wordt in o.a. home-assistant en dsmr-reader.

Deze map wordt nooit zelf gedeployed (geen eigen ArgoCD Application, geen
`overlay/`) — hij bestaat puur om vanuit de `kustomization.yaml` van een
andere app te worden meegenomen via `resources:`.

## Belangrijk: kustomize hernoemt de SealedSecret niet automatisch mee

`namePrefix` hernoemt alle resources en fixt ook automatisch de meeste
onderlinge verwijzingen (Service ↔ StatefulSet `serviceName`, ConfigMap ↔
`envFrom`). Dat automatische fixen kent kustomize alleen voor een vaste lijst
ingebouwde kinds — en `SealedSecret` staat daar niet bij (in tegenstelling
tot een gewone `Secret`). Zonder ingrijpen krijg je dus een StatefulSet die
naar het verkeerde (niet-prefixed) secret-record verwijst.

Om dezelfde reden werkt de `labels:`-transformer met `includeSelectors: true`
niet door tot in de `volumeClaimTemplates`-selector van de StatefulSet — die
zit daarvoor te diep genest. Vandaar dat de PVC hieronder aan een PV wordt
gepind via een expliciete `volumeName`, niet via een label-selector.

Elke app die deze base gebruikt heeft daarom **altijd** deze twee extra
patches nodig (zie het volledige voorbeeld hieronder): de SealedSecret's
eigen `spec.template.metadata.name`, en de StatefulSet's
`secretKeyRef.name` + `volumeClaimTemplates[0].spec.volumeName`.

Dit is echt getest met `kubectl kustomize` (niet alleen opgeschreven) — zie
onderstaand voorbeeld, dat 1:1 is uitgeprobeerd.

## Ook belangrijk: het gedeelde `app`-label

De resources in deze base delen allemaal het label `app: postgresql-database`.
Gebruik je deze base voor twee apps in dezelfde namespace (zoals `tools`,
waar meerdere apps samen draaien), dan matchen hun selectors elkaar zonder
een extra, per-app uniek label — de ene app zou dan de pods van de andere
als "eigen" kunnen zien. Voeg daarom **altijd** een eigen `app-instance`-label
toe via de `labels:` transformer, zoals hieronder.

## Wat je zelf nog moet doen buiten Kubernetes

- Op TrueNAS zelf een nieuwe iSCSI-target/LUN aanmaken voor deze app (net als
  voor elke bestaande postgres-instance) en de bijbehorende IQN in de patch
  hieronder zetten.
- Een echt wachtwoord verzegelen met `kubeseal` voor de uiteindelijke naam
  van je secret (`<prefix>-postgresql-secret`) en namespace, en de
  `encryptedData.POSTGRES_PASSWORD` placeholder daarmee vervangen. Zonder dit
  blijft de placeholder-waarde staan en start postgres niet.

## Volledig voorbeeld

`manifests/mijnapp/base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../postgres-base/base
  - manifest.yaml # de app zelf (Deployment/Service/HTTPRoute etc.)

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
      - op: replace
        path: /spec/volumeClaimTemplates/0/spec/volumeName
        value: mijnapp-pv-iscsi-postgresql-db
      - op: replace
        path: /spec/template/spec/containers/0/env/0/valueFrom/secretKeyRef/name
        value: mijnapp-postgresql-secret
  - target:
      kind: SealedSecret
      name: postgresql-secret
    patch: |-
      - op: replace
        path: /spec/template/metadata/name
        value: mijnapp-postgresql-secret
  - target:
      kind: PersistentVolume
      name: pv-iscsi-postgresql-db
    patch: |-
      - op: replace
        path: /spec/capacity/storage
        value: 20Gi
      - op: replace
        path: /spec/csi/volumeAttributes/iqn
        value: "iqn.2005-10.org.freenas.ctl:mijnapp-db"
```

Na `namePrefix` heten de resources dan `mijnapp-postgresql` (StatefulSet),
`mijnapp-postgresql-hl` (Service), `mijnapp-postgresql-env` (ConfigMap),
`mijnapp-pv-iscsi-postgresql-db` (PV) en `mijnapp-postgresql-secret`
(SealedSecret) — allemaal automatisch consistent dankzij de patches hierboven.
Je app verbindt met `mijnapp-postgresql-hl:5432`.

### Andere postgres-versie of extension (zoals immich's pgvector-image)?

De base draait standaard op `docker.io/postgres:18-alpine`, met de data-volume
gemount op `/var/lib/postgresql` (niet `/var/lib/postgresql/data` — sinds
postgres 18 zet het image `PGDATA` in een versie-specifieke submap,
`/var/lib/postgresql/<major>/docker`, en is de `VOLUME` van het image
`/var/lib/postgresql` geworden). Wil je een andere versie of een extension-
image (zoals immich's pgvector-variant), patch dan zowel het image als de
mountPath:

```yaml
  - target:
      kind: StatefulSet
      name: postgresql
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: docker.io/postgres:16-alpine
      - op: replace
        path: /spec/template/spec/containers/0/volumeMounts/1/mountPath
        value: /var/lib/postgresql/data
```

Let op: postgres 16 en ouder (en de meeste extension-images die daarop
gebaseerd zijn, zoals immich's) gebruiken nog de oude `/var/lib/postgresql/data`
als data-directory — check de image-documentatie van de gekozen versie/image
voordat je de mountPath ongewijzigd laat staan.

## Bestaande apps migreren?

Dit is bewust niet gedaan voor home-assistant, dsmr-reader of
open-family-finance — hun StatefulSets draaien al met bestaande data.
Migreren kan, maar is een aparte, voorzichtige actie (PVC-namen/labels
veranderen bij een StatefulSet raakt al snel de bestaande PV-binding) en
geen automatisch gevolg van het toevoegen van deze base.
