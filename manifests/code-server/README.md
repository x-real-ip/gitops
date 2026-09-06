# code-server

VS Code in de browser, bereikbaar op <https://code.lan.stamx.nl> via de private
gateway. Draait in namespace `tools`.

## Wat je zelf nog moet doen buiten Kubernetes

- Op TrueNAS een iSCSI-target/LUN aanmaken met IQN
  `iqn.2005-10.org.freenas.ctl:code-server-data` (20Gi). Die wordt gemount op
  `/home/coder`, dus daar staan je instellingen, extensies en werkmappen.
- Een echt wachtwoord verzegelen met `kubeseal` en de `PASSWORD` placeholder in
  `base/sealedsecret.yaml` vervangen. Zonder dit blijft de placeholder staan,
  kan de sealed-secrets controller het secret niet uitpakken en start de pod
  niet.

  ```sh
  kubectl create secret generic code-server-secret \
    --namespace tools \
    --from-literal=PASSWORD='<jouw-wachtwoord>' \
    --dry-run=client -o yaml \
    | kubeseal --format yaml
  ```

  Neem uit de output de waarde van `spec.encryptedData.PASSWORD` over.

## Versiebeheer

De image-tag staat in `overlay/kustomization.yaml` en wordt door de
`Check releases` workflow gebumpt (matrix-entry `code-server`, type `image` op
`ghcr.io/coder/code-server`). Alleen kale versietags worden meegenomen; de
distro-varianten (`-debian`, `-ubuntu`, `-fedora`, ...) en `latest` vallen af
via `include_tags: "^[0-9][0-9.]*$"`.
