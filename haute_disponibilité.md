# Haute disponibilité d'un service Web dynamique (Corosync / Pacemaker / MySQL)

> Documentation technique basée sur les travaux du CERTA (Septembre 2016).

## Sommaire

- [1. Principe de la haute disponibilité](#1-principe-de-la-haute-disponibilité)
- [2. Architecture](#2-architecture)
- [3. Installation et configuration de Corosync/Pacemaker](#3-installation-et-configuration-de-corosyncpacemaker)
- [4. IP virtuelle (failover IP)](#4-ip-virtuelle-failover-ip)
- [5. Service Web en haute disponibilité](#5-service-web-en-haute-disponibilité)
- [6. Réplication MySQL maître-esclave](#6-réplication-mysql-maître-esclave)
- [7. Réplication multi-maître et intégration au cluster](#7-réplication-multi-maître-et-intégration-au-cluster)
- [8. Commandes utiles](#8-commandes-utiles)

---

## 1. Principe de la haute disponibilité

La **haute disponibilité** (HA) vise à garantir un taux de disponibilité élevé d'un service (99 %, 99,9 %, 99,99 %...). Le modèle **actif/passif** à deux nœuds repose sur :

- un **serveur secondaire**, configuré à l'identique du serveur primaire, qui surveille ce dernier en permanence ;
- une bascule automatique (**failover**) si le serveur primaire tombe en panne.

### Le heartbeat

- Le serveur actif émet régulièrement des signaux réseau pour signaler qu'il est vivant.
- Le serveur passif écoute passivement.
- Si plus aucun signal n'arrive, le passif se considère comme actif, bascule les services et se met à émettre ses propres battements.
- Au retour du serveur d'origine, celui-ci reprend, dans un premier temps, le rôle de nœud passif.

### Les outils

| Outil | Rôle |
|---|---|
| **Corosync** | Détecte la défaillance d'un nœud et gère l'infrastructure du cluster (état des nœuds, communication). |
| **Pacemaker** | Gestionnaire de ressources : crée, démarre, arrête et supervise les services du cluster. |

---

## 2. Architecture

| Serveur | Rôle | IP réelle | IP virtuelle |
|---|---|---|---|
| `serv1` | Maître | 172.16.0.10 | 172.16.0.12 |
| `serv2` | Esclave | 172.16.0.11 | 172.16.0.12 |

```
                     ┌────────────────────┐
        Client ───►  │   IP virtuelle      │
                     │   172.16.0.12       │
                     └─────────┬──────────┘
                 ┌─────────────┴─────────────┐
        ┌────────▼────────┐         ┌────────▼────────┐
        │     serv1        │         │     serv2        │
        │  172.16.0.10     │ heartbeat  172.16.0.11     │
        │  Corosync +      │◄────────►│  Corosync +      │
        │  Pacemaker +     │         │  Pacemaker +     │
        │  Apache + MySQL  │         │  Apache + MySQL  │
        │  (réplication multi-maître entre les deux)     │
        └──────────────────┘         └──────────────────┘
```

---

## 3. Installation et configuration de Corosync/Pacemaker

### Installation

```bash
apt update
apt install corosync pacemaker crmsh
```

### Clé d'authentification

Génère une clé partagée par tous les nœuds, servant aussi à chiffrer les échanges :

```bash
corosync-keygen
```

Le fichier `/etc/corosync/authkey` doit être **identique sur tous les nœuds** (propagé ici par clonage de la VM).

### Fichier `/etc/corosync/corosync.conf`

```ini
totem {
    version: 2
    cluster_name: cluster_web      # ⚠️ À ADAPTER : nom du cluster, libre
    crypto_cipher: aes256
    crypto_hash: sha1
    clear_node_high_bit: yes
}

logging {
    fileline: off
    to_logfile: yes
    logfile: /var/log/corosync/corosync.log   # ⚠️ À ADAPTER si vous voulez un autre emplacement de log
    to_syslog: no
    debug: off
    timestamp: on
}

quorum {
    provider: corosync_votequorum
    expected_votes: 2              # ⚠️ À ADAPTER : nombre de votes attendus = nombre de nœuds du cluster
    two_nodes: 1
}

nodelist {
    node {
        name: serv1                # ⚠️ À ADAPTER : nom d'hôte du nœud maître
        nodeid: 1                  # ⚠️ À ADAPTER : identifiant unique du nœud (doit différer entre nœuds)
        ring0_addr: 172.16.0.10    # ⚠️ À ADAPTER : adresse IP réelle du nœud maître
    }
    node {
        name: serv2                # ⚠️ À ADAPTER : nom d'hôte du nœud esclave
        nodeid: 2                  # ⚠️ À ADAPTER : identifiant unique du nœud (doit différer entre nœuds)
        ring0_addr: 172.16.0.11    # ⚠️ À ADAPTER : adresse IP réelle du nœud esclave
    }
}

service {
    ver: 0
    name: pacemaker
}
```

### Désactivation de STONITH et du quorum

**STONITH** (*Shoot The Other Node In The Head*) éteint proprement un nœud défaillant — utile surtout avec des disques partagés. Avec un cluster à seulement 2 nœuds, le quorum est perdu dès qu'un nœud tombe : il faut demander à Pacemaker de l'ignorer.

```bash
crm configure property stonith-enabled=false
crm configure property no-quorum-policy="ignore"
```

Vérification :

```bash
crm_verify -L -V
crm configure show
```

> ⚠️ Ne jamais modifier `/var/lib/pacemaker/cib/cib.xml` directement : toujours passer par les commandes `crm`.

---

## 4. IP virtuelle (failover IP)

Le client ne doit connaître qu'**une seule adresse IP**, rattachée au cluster et non à une machine précise.

```bash
crm configure primitive IPFailover ocf:heartbeat:IPaddr2 \
    params ip=172.16.0.12 cidr_netmask=24 nic=eth0 iflabel=VIP \
    op monitor interval="10s" timeout="30s" \
    op stop timeout="40s"
```

| Paramètre | ⚠️ À adapter ? | Détail |
|---|---|---|
| `ip=172.16.0.12` | ✅ Oui | L'adresse IP virtuelle du cluster |
| `cidr_netmask=24` | ✅ Oui | Le masque de sous-réseau, selon votre réseau |
| `nic=eth0` | ✅ Oui | Le nom de l'interface réseau (`ip a` pour la connaître) |
| `iflabel=VIP` | Libre | Juste un label, peut rester tel quel |
| `interval` / `timeout` | Optionnel | Peut rester par défaut, à affiner selon vos besoins |

Préférence de nœud (contrainte de localisation) :

```bash
crm resource move IPFailover serv1   # ⚠️ serv1 À ADAPTER : nom du nœud vers lequel migrer la ressource
```

Empêcher le retour automatique après une panne :

```bash
crm configure property default-resource-stickiness=100
```

Test :

```bash
crm node standby serv1   # ⚠️ serv1 À ADAPTER selon le nœud à mettre en maintenance
crm node online serv1    # ⚠️ serv1 À ADAPTER selon le nœud à remettre en ligne
```

---

## 5. Service Web en haute disponibilité

```bash
crm configure

# ⚠️ lsb:apache2 À ADAPTER si votre distribution nomme le service différemment (ex: httpd)
primitive serviceWeb lsb:apache2 \
    op monitor interval=60s \
    op start interval=0 timeout=60s \
    op stop interval=0 timeout=60s

commit
quit
```

Par défaut, Pacemaker peut répartir `IPFailover` et `serviceWeb` sur des nœuds différents. Pour les forcer sur le même nœud, on les regroupe :

```bash
crm configure

# ⚠️ migration-threshold=5 optionnel : ajustez le nombre d'échecs tolérés avant migration définitive
group servweb IPFailover serviceWeb meta migration-threshold="5"

commit
quit
```

> `migration-threshold` = nombre d'échecs tolérés avant migration définitive du groupe vers l'autre nœud.

En cas de blocage après plusieurs erreurs :

```bash
crm resource cleanup <id_ressource>
```

---

## 6. Réplication MySQL maître-esclave

### Principe

- Le **maître** journalise toutes les écritures (`INSERT`, `UPDATE`, `DELETE`) dans un **log binaire**.
- L'**esclave** se connecte au maître, lit ce journal depuis une position donnée, et rejoue les requêtes localement.

> Les bases doivent être **identiques** sur les deux serveurs avant de démarrer la réplication.

### Configuration maître (`/etc/mysql/mariadb.conf.d/50-server.cnf`)

```ini
[mysqld]
#bind-address = 127.0.0.1
server-id = 100                        # ⚠️ À ADAPTER : identifiant unique du maître (doit différer de l'esclave)
log_bin = /var/log/mysql/mysql-bin.log
expire_logs_days = 10
max_binlog_size = 100M
binlog_do_db = nom_de_la_base          # ⚠️ À ADAPTER : nom réel de la base à répliquer
```

### Configuration esclave

```ini
[mysqld]
server-id = 104                 # ⚠️ À ADAPTER : identifiant unique de l'esclave (différent du maître)
expire_logs_days = 10
max_binlog_size = 100M
master-retry-count = 20
replicate-do-db = nom_de_la_base   # ⚠️ À ADAPTER : nom réel de la base à répliquer (identique au maître)
```

### Démarrage de la réplication

Sur le maître :

```sql
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;           -- ⚠️ note File et Position affichés : à réutiliser tels quels dans CHANGE MASTER TO
```

Sur l'esclave :

```sql
STOP SLAVE;

CHANGE MASTER TO
    MASTER_HOST='172.16.0.10',        -- ⚠️ À ADAPTER : IP réelle du serveur maître
    MASTER_USER='replicateur',        -- ⚠️ À ADAPTER : nom de l'utilisateur MySQL de réplication créé sur le maître
    MASTER_PASSWORD='mdpreplicateur', -- ⚠️ À ADAPTER : mot de passe de cet utilisateur
    MASTER_LOG_FILE='mysql-bin.000001', -- ⚠️ À ADAPTER : fichier binlog relevé via SHOW MASTER STATUS
    MASTER_LOG_POS=3921;                -- ⚠️ À ADAPTER : position relevée via SHOW MASTER STATUS

START SLAVE;
```

Sur le maître :

```sql
UNLOCK TABLES;
```

Vérification :

```sql
SHOW SLAVE STATUS \G;
```

### Dépannage courant

| Problème | Cause | Solution |
|---|---|---|
| `Duplicate entry ... Error_code: 1062` | Bases non synchronisées au départ | Dump du maître, restauration sur l'esclave, relancer `CHANGE MASTER TO` |
| Décalage `File`/`Position` entre maître et esclave | Écriture faite par erreur sur l'esclave | Resynchroniser les bases |
| Accumulation des binlogs | Pas de purge | `PURGE MASTER LOGS TO 'mysql-bin.NNNNNN';` (supprime tous les binlogs antérieurs au fichier indiqué) |

---

## 7. Réplication multi-maître et intégration au cluster

### Problématique

Avec une réplication simple maître-esclave, si le maître tombe puis revient, Pacemaker lui redonne la priorité — mais il n'a pas connaissance des données écrites sur l'esclave entre-temps. Risque de perte de données.

### Solution : réplication multi-maître

Chaque serveur est à la fois **maître** et **esclave** de l'autre (deux réplications croisées). Les données restent à jour sur les deux nœuds, quel que soit celui qui est actif.

Étapes :

1. Compléter la configuration de chaque nœud avec ce qui lui manque (config esclave sur le maître actuel, config maître sur l'esclave actuel).
2. Créer l'utilisateur MySQL de réplication sur le « nouveau maître ».
3. Redémarrer MySQL des deux côtés.
4. Relever binlog et position sur chaque nœud, puis exécuter `CHANGE MASTER TO` + `START SLAVE` sur le nœud qui était maître.

> ⚠️ Ajouter **`log-slave-updates`** sur chaque serveur pour éviter qu'un événement déjà exécuté ne soit rejoué en boucle.

Vérifier sur les **deux** serveurs :

```sql
SHOW SLAVE STATUS \G;
```

Pour une répartition de charge (éviter les conflits de clés auto-incrémentées) :

```ini
auto_increment_increment = 2   ; ⚠️ À ADAPTER : N = nombre total de maîtres (2 ici, plus si multi-maître étendu)
auto_increment_offset = 1      ; ⚠️ À ADAPTER : valeur unique par maître (1 sur serv1, 2 sur serv2, etc.)
```

### Intégration de MySQL dans Pacemaker (mode actif/actif)

```bash
# ⚠️ socket À ADAPTER si votre installation MySQL/MariaDB utilise un autre chemin de socket
crm configure primitive serviceMySQL ocf:heartbeat:mysql \
    params socket="/var/run/mysqld/mysqld.sock"

crm configure clone cServiceMySQL serviceMySQL
```

Un **clone anonyme** (type par défaut) démarre la ressource simultanément et à l'identique sur les deux nœuds — Pacemaker ne désactive jamais MySQL sur aucun des deux nœuds ; c'est la réplication multi-maître qui assure la cohérence.

Résultat attendu (`crm_mon`) :

```
2 nodes and 4 resources configured
Online: [ serv2 serv1 ]
Resource Group: servIntralab
    IPFailover     (ocf::heartbeat:IPaddr2): Started serv1
    serviceWeb     (ocf::heartbeat:apache):  Started serv1
Clone Set: cServiceMySQL [serviceMySQL]
    Started: [ serv2 serv1 ]
```

---

## 8. Commandes utiles

```bash
/etc/init.d/corosync status
crm_mon
crm status

crm configure show
crm configure edit
crm configure edit <id>
crm_verify -L -V

crm resource stop <id>            # ⚠️ <id> À REMPLACER par le nom réel de la ressource
crm resource start <id>           # ⚠️ <id> À REMPLACER par le nom réel de la ressource
crm resource move <id> <noeud>    # ⚠️ <id> et <noeud> À REMPLACER
crm resource unmove <id>          # ⚠️ <id> À REMPLACER
crm resource cleanup <id>         # ⚠️ <id> À REMPLACER
crm configure delete <id>         # ⚠️ <id> À REMPLACER

crm configure location <nouvel_id> <id_ressource> <score>: <noeud>
# ⚠️ Tous les <...> À REMPLACER : nom de la contrainte, ressource visée, score de préférence, nœud cible

crm ra list ocf heartbeat
crm ra info ocf:heartbeat:<nom_primitive>    # ⚠️ <nom_primitive> À REMPLACER (ex: IPaddr2, apache, mysql)

crm node standby [<noeud>]   # ⚠️ <noeud> À REMPLACER
crm node online [<noeud>]    # ⚠️ <noeud> À REMPLACER

crm configure clone <nom_clone> <id_ressource>   # ⚠️ <nom_clone> et <id_ressource> À REMPLACER

crm configure save /root/sauveha.conf          # ⚠️ chemin optionnel à adapter selon votre organisation
crm configure load replace /root/sauveha.conf  # ⚠️ chemin optionnel à adapter selon votre organisation
```

```sql
SHOW MASTER STATUS;
SHOW SLAVE STATUS \G;
STOP SLAVE;
START SLAVE;

-- ⚠️ Toutes les valeurs '...' À REMPLACER par vos informations réelles
CHANGE MASTER TO MASTER_HOST='...', MASTER_USER='...', MASTER_PASSWORD='...',
                  MASTER_LOG_FILE='...', MASTER_LOG_POS=...;

FLUSH TABLES WITH READ LOCK;
UNLOCK TABLES;

PURGE MASTER LOGS TO 'mysql-bin.NNNNNN';   -- ⚠️ 'mysql-bin.NNNNNN' À REMPLACER par le nom du fichier binlog courant
```

---

## Source

CERTA — *Haute disponibilité d'un service Web dynamique*, Septembre 2016, v2.0 — [reseaucerta.org](http://www.reseaucerta.org)
