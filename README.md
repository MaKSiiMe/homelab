# homelab

Infrastructure du serveur `s56c-srv` (Debian 13, Docker + Portainer CE), gérée en GitOps : chaque dossier est une stack Git Portainer qui suit la branche `main` (polling 5 min).

| Dossier | Stack | Contenu |
|---|---|---|
| `monitoring/` | `monitoring` | Uptime Kuma, Dozzle, socket-proxy Docker en lecture seule |
| `metrics/` | `metrics` | Prometheus, Alertmanager, Grafana, node_exporter, cAdvisor, smartctl_exporter |

Les secrets ne sont jamais dans ce dépôt : ce sont des variables de la stack dans Portainer.

## Qui surveille quoi

| Question | Outil | Alerte |
|---|---|---|
| Le conteneur / le service répond-il ? | Uptime Kuma | Discord `#alertes` |
| Les ressources tiennent-elles (RAM, disque, CPU, température, SSD) ? | Prometheus → Alertmanager | Discord `#alertes` |
| Kuma est-il vivant ? | cron → healthchecks.io | Discord (healthchecks.io) |
| Prometheus et Alertmanager sont-ils vivants ? | alerte `Watchdog` → healthchecks.io | Discord (healthchecks.io) |

## Limites mémoire : pas de swap

Tout service avec un `mem_limit` doit avoir un `memswap_limit` **égal** (la CI le vérifie). Sans `memswap_limit`, Docker autorise un swap égal à la limite (`memswap_limit` = 2 × `mem_limit`), et ce swap est sur le SSD. Un conteneur trop serré n'est alors pas tué : il swappe et relit son cache en boucle, ce qui use le disque sans qu'on le voie. Exemple : grafana le 18/11, avec une limite de 256m, avait ~124 Mo en swap, et avait lu 91 Go et écrit 39 Go en 17 h, sans aucun utilisateur. Sans swap, la limite est atteinte franchement : OOM kill, signalé par l'alerte `ConteneurOomKill`, et on augmente `mem_limit`.

## Stack `metrics` : règles d'édition

Portainer CE ne monte pas les fichiers du dépôt (chemins relatifs réservés à la Business Edition). Les configurations sont donc écrites en ligne dans `metrics/compose.yml` (`configs: content`) et copiées dans les conteneurs à leur création.

1. **`$$` au lieu de `$`** dans les blocs `content` : Compose interpole les variables (`{{ $$labels.job }}`, `{{ $$value }}`).
2. **Incrémenter `homelab.config-rev`** après toute modification d'une config : Compose ne recrée pas un conteneur quand seul le contenu d'une config change.
3. La CI (`.github/workflows/validate.yml`) rend le compose et passe les configs à `promtool` et `amtool`. Ne pas fusionner si elle est rouge : Portainer déploierait la config cassée.

**Variables Portainer de la stack `metrics`** : `DISCORD_WEBHOOK_URL`, `HEALTHCHECKS_WATCHDOG_URL`, `GRAFANA_ADMIN_PASSWORD` (ce dernier n'est lu qu'à la création de la base Grafana). Elles sont injectées par interpolation (`${VAR:?}`) dans des configs montées sous `/run/secrets/` : `secrets: environment:` ne reçoit pas les variables Portainer. Après avoir changé l'une d'elles, incrémenter `homelab.config-rev`.

**Prérequis sur l'hôte**
- `net.ipv4.ip_nonlocal_bind = 1` (déjà en place pour la stack `monitoring`).
- node_exporter tourne dans le réseau de l'hôte et écoute sur `10.203.0.1:9100`, la passerelle du pont `br-metrics`. UFW doit laisser passer Prometheus :
  ```bash
  sudo ufw allow in on br-metrics to 10.203.0.1 port 9100 proto tcp comment 'Prometheus -> node_exporter'
  ```
- La stack `monitoring` doit tourner (cAdvisor passe par son socket-proxy, réseau `monitoring_socket`).
- cAdvisor monte `/run/containerd/containerd.sock` : Docker utilise le stockage d'images de containerd, et sans ce socket cAdvisor ne nomme aucun conteneur. Ce socket équivaut à un accès root ; compromis accepté (aucun port publié, image épinglée).

**Accès (Tailscale uniquement)** : Grafana `:3000`, Prometheus `:9090`, Alertmanager `:9093` sur `100.124.153.20`.
