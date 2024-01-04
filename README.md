# GitOps

- [GitOps](#gitops)
  - [Start ArgoCD WebUI](#start-argocd-webui)
  - [Apps](#apps)
    - [mosquitto](#mosquitto)
    - [media-entertainment](#media-entertainment)

## Start ArgoCD WebUI

```bash
argocd admin dashboard -n argocd
```

## Apps

### mosquitto

- [mosquitto](https://mosquitto.org/): an open source message broker that
  implements the MQTT protocol.

### media-entertainment

- [Transmission](https://transmissionbt.com/): Torrent download client with
  OpenVPN where Transmission is running only when OpenVPN has an active tunnel.
- [Plex](https://www.plex.tv): a multimedia applications designed to organize,
  manage, and stream digital media files to networked devices.
- [Overseerr](https://overseerr.dev/): a free and open source software
  application for managing requests for your media library.
- [Prowlarr](https://github.com/Prowlarr/Prowlarr/): Prowlarr is an indexer
  manager/proxy base stack offering complete management of torrent indexers with
  no per app indexer setup required.
- [Bazarr](https://www.bazarr.media/): Bazarr is a companion application to
  Sonarr and Radarr that manages and downloads subtitles.
- [Radarr](https://radarr.video/): Radarr is a movie collection manager to grab,
  sort, and rename them. It can also be configured to automatically upgrade the
  quality of existing files in the library when a better quality format becomes
  available.
- [Sonarr](https://sonarr.tv/): Sonarr is a TV show and series collection
  manager. It can monitor for new episodes of shows and series and will grab,
  sort and rename them. It can also be configured to automatically upgrade the
  quality of files already downloaded when a better quality format becomes
  available.
