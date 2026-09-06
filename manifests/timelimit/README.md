# TimeLimit.io Server

Server voor de "connected mode" van de TimeLimit-app, bereikbaar op
<https://timelimit.lan.stamx.nl>.

De image komt uit de fork die ook de officiele helm-chart publiceert
(`ghcr.io/michaelsp/timelimit-server/timelimit`). Die fork leest de database
niet uit een `DATABASE_URL`, maar bouwt de connectiestring zelf op uit
`DB_DRIVER`, `DB_USER`, `DB_PASS`, `DB_HOST`, `DB_PORT` en `DB_NAME`. De docs
van het upstream-project noemen nog `DATABASE_URL`; die is hier niet in
gebruik.

## Opslag

De app zelf schrijft niets naar disk: alle gezinnen, apparaten, categorieen en
tijdregels staan in postgres. Persistentie loopt dus volledig via de gedeelde
`manifests/postgres/base` (iSCSI-PV van 5Gi, `Retain`). Een herstart,
reschedule of image-update van de pod raakt geen data.

## Wat je zelf nog moet doen buiten Kubernetes

- Op TrueNAS een iSCSI-target/LUN aanmaken met IQN
  `iqn.2005-10.org.freenas.ctl:timelimit-db` (5Gi), net als voor elke andere
  postgres-instance.
- Een DNS-record `timelimit.lan.stamx.nl` naar de private gateway
  (`10.33.33.100`) laten wijzen.
- Twee secrets verzegelen met `kubeseal`, allebei in namespace `tools`:

  ```sh
  # Het databasewachtwoord. Gebruik alleen letters en cijfers: de server plakt
  # deze waarde ongecodeerd in een connectie-URL.
  DB_PASS="$(openssl rand -hex 24)"

  # 1. wachtwoord voor postgres zelf
  kubectl create secret generic timelimit-postgresql-secret \
    --namespace tools --dry-run=client -o yaml \
    --from-literal=POSTGRES_PASSWORD="$DB_PASS" \
  | kubeseal --scope cluster-wide --format yaml

  # 2. secrets voor de applicatie
  kubectl create secret generic timelimit-secrets \
    --namespace tools --dry-run=client -o yaml \
    --from-literal=DB_PASS="$DB_PASS" \
    --from-literal=ADMIN_TOKEN="$(openssl rand -hex 24)" \
    --from-literal=SIGN_SECRET="$(openssl rand -hex 32)" \
  | kubeseal --scope cluster-wide --format yaml
  ```

  Zet de `encryptedData`-waarden uit stap 1 in de `SealedSecret`-patch in
  `base/postgresql/kustomization.yaml` (zoals bij depart en authentik) en die
  uit stap 2 in `base/sealedsecret.yaml`. Zolang de placeholders blijven staan
  maakt de sealed-secrets controller de Secrets niet aan en start de pod niet.

## Na de eerste sync

- Maak je gezin aan in de app (aanmelden gaat via een mail die de server
  verstuurt via de protonmail-bridge in dezelfde namespace).
- Zet daarna `DISABLE_SIGNUP` in `base/configmap.yaml` op `"yes"`, zodat er
  geen nieuwe gezinnen meer op deze server aangemaakt kunnen worden.

## Let op

De gepubliceerde image bestaat alleen voor `linux/amd64`. De odroid-nodes in
het cluster staan op `machine_type=odroid:NoSchedule` en deze deployment zet
geen toleration, dus de pod landt vanzelf op een amd64-node.

De container draait met een read-only rootfilesystem als uid/gid 1000, met
schrijfbare emptyDirs op `/home/node` en `/tmp`. Dat is genoeg omdat de image
al gebouwd is (`npm start` doet alleen `node ./build/index.js`) en de app niets
naar disk schrijft. Mocht npm er bij een toekomstige image-versie toch over
struikelen, dan is `readOnlyRootFilesystem: false` in `base/deployment.yaml` de
enige aanpassing die nodig is.
