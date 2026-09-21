# Haute disponibilité d'un service web — GSB

*Documentation de mise en œuvre du TP B2-Act2-TP1 (contexte GSB) — cluster actif/passif Corosync + Pacemaker avec réplication MariaDB multi-maître.*

*Document unique issu de la fusion du support CERTA 2016, de la version corrigée 1.1 (Debian 12/13, Corosync 3.x, Pacemaker 2.x, crmsh 4.x, MariaDB 10.11/11.x) et des relevés de commandes du TP. Les commandes de 2016 qui ne fonctionnent plus ne sont plus reproduites : elles sont recensées dans le tableau des corrections (§9).*

---

## Sommaire

1. [Principe et architecture](#1-principe-et-architecture)
2. [Prérequis communs aux deux nœuds](#2-prérequis-communs-aux-deux-nœuds)
3. [Activité 1 — Corosync et Pacemaker](#3-activité-1--corosync-et-pacemaker)
4. [Activité 2 — Ressources IPFailover et serviceWeb](#4-activité-2--ressources-ipfailover-et-serviceweb)
5. [Activité 3 — Réplication MariaDB maître-esclave](#5-activité-3--réplication-mariadb-maître-esclave)
6. [Activité 4 — Multi-maître et intégration au cluster](#6-activité-4--multi-maître-et-intégration-au-cluster)
7. [Tests de validation](#7-tests-de-validation)
8. [Dépannage](#8-dépannage)
9. [Tableau des corrections 2016 → aujourd'hui](#9-tableau-des-corrections-2016--aujourdhui)
10. [Aide-mémoire des commandes](#10-aide-mémoire-des-commandes)
11. [Sources de vérification](#11-sources-de-vérification)

---

## 1. Principe et architecture

### 1.1 Principe de la haute disponibilité

La haute disponibilité (HA) vise à garantir un taux de disponibilité élevé d'un service (99 %, 99,9 %, 99,99 %…). Le modèle **actif/passif à deux nœuds** repose sur :

- un serveur secondaire configuré à l'identique du primaire, qui le surveille en permanence ;
- une bascule automatique (*failover*) si le primaire tombe en panne.

**Le heartbeat** : le nœud actif émet régulièrement des signaux réseau pour signaler qu'il est vivant ; le passif écoute. Si plus aucun signal n'arrive, le passif se considère comme actif, bascule les services et se met à émettre ses propres battements. Au retour du serveur d'origine, celui-ci reprend d'abord le rôle de nœud passif.

| Outil | Rôle |
|---|---|
| **Corosync 3** | Couche cluster : détecte la défaillance d'un nœud, gère la communication et le quorum. |
| **Pacemaker 2** | Gestionnaire de ressources : crée, démarre, arrête et supervise les services du cluster. |
| **crmsh** | Interface en ligne de commande (`crm`). Alternative : `pcs`, mais le TP est écrit pour `crm`. |

En actif/passif, le service est **installé sur les deux nœuds mais démarré uniquement sur le nœud actif**.

### 1.2 Architecture du TP

| Élément | serv1 (maître) | serv2 (esclave) |
|---|---|---|
| IP réelle | `172.16.0.10` | `172.16.0.11` |
| IP virtuelle (VIP) | `172.16.0.12/24` | `172.16.0.12/24` |
| `server_id` MariaDB | `1` | `2` |
| Nom d'hôte | `serv1` | `serv2` |

- **Base répliquée** : `gsb_valide`
- **Compte de réplication** : `replicateur` / `Btssio2017`
- **Services en HA** : Apache2 (actif/passif), MariaDB (actif/actif cloné)
- **Interface réseau** : à relever avec `ip -br link` (`ens33`, `ens18`, `enp0s3`… — plus jamais `eth0`)

> Les noms déclarés dans `corosync.conf` doivent être **exactement** les noms d'hôte des machines.

                 ┌─────────────────────┐
    Client ───►  │    IP virtuelle     │
                 │    172.16.0.12      │
                 └──────────┬──────────┘
             ┌──────────────┴──────────────┐
    ┌────────▼────────┐           ┌────────▼────────┐
    │      serv1      │           │      serv2      │
    │   172.16.0.10   │ heartbeat │   172.16.0.11   │
    │   Corosync +    │◄─────────►│   Corosync +    │
    │   Pacemaker +   │           │   Pacemaker +   │
    │ Apache + MariaDB│           │ Apache + MariaDB│
    │   (réplication multi-maître entre les deux)   │
    └─────────────────┘           └─────────────────┘

    
---

## 2. Prérequis communs aux deux nœuds

### 2.1 Nom d'hôte

```bash
hostnamectl set-hostname serv1     # serv2 sur l'autre nœud
```

Définit le nom d'hôte de façon persistante. Ce nom doit correspondre exactement au champ `name:` de `corosync.conf`.

### 2.2 Résolution de noms

```bash
printf '%s\n' "172.16.0.10 serv1" "172.16.0.11 serv2" | tee -a /etc/hosts
```

Indispensable : le cluster ne doit pas dépendre d'un DNS externe pour se former.

### 2.3 Identifier le nom réel de l'interface réseau

```bash
ip -br link
```

Debian utilise les noms prédictibles (`ens33`, `ens18`, `enp0s3`…). Ce nom sera réutilisé dans le paramètre `nic=` de la VIP.

### 2.4 Vérifier l'application avant clustering

```bash
curl -I http://172.16.0.10/applifrais/
```

Vérifie que l'application de gestion de frais répond en HTTP avant d'ajouter la moindre couche HA (`HTTP/1.1 200 OK` attendu).

### 2.5 Ouvrir les ports du cluster (si pare-feu actif)

```bash
ufw allow from 172.16.0.11 to any port 5405 proto udp
```

Corosync 3 (transport `knet`) échange sur **UDP/5405** entre les nœuds. Sans cette règle, le cluster ne se forme pas et chaque nœud se croit seul.

---

## 3. Activité 1 — Corosync et Pacemaker

### 3.1 Installation

```bash
apt update
apt install -y corosync pacemaker pacemaker-cli-utils crmsh fence-agents
```

`pacemaker-cli-utils` fournit `crm_mon`, `crm_verify`, `crm_resource` ; `crmsh` fournit `crm` ; `fence-agents` sert pour le STONITH en production.

### 3.2 Désactiver le démarrage automatique des services gérés par le cluster

```bash
systemctl disable --now apache2
systemctl disable --now mariadb
```

> ⚠️ **Étape critique, absente du support d'origine.** Tout service confié à Pacemaker doit être retiré du démarrage systemd, sinon les deux se disputent le contrôle : Apache tournerait aussi sur le nœud passif, et MariaDB serait démarrée avant que le cluster ne la sonde. C'est le cluster, et lui seul, qui démarre ces services.
>
> MariaDB sera configurée en clone actif/actif (§6.4) : elle tournera bien sur les deux nœuds, mais démarrée par Pacemaker, pas par systemd.

### 3.3 Génération de la clé d'authentification

```bash
corosync-keygen
```

Génère `/etc/corosync/authkey` (256 octets, droits `0400`), qui authentifie et chiffre les échanges entre nœuds.

> **Correction 2016** : il n'y a plus à « taper au clavier pour générer de l'entropie » — depuis Corosync 2.x la clé est lue sur `/dev/urandom`, c'est instantané.

### 3.4 Copie de la clé sur l'autre nœud

```bash
scp /etc/corosync/authkey root@172.16.0.11:/etc/corosync/authkey
ssh root@172.16.0.11 "chown root:root /etc/corosync/authkey && chmod 400 /etc/corosync/authkey"
```

La clé doit être identique sur tous les nœuds. **Si serv2 est issu d'un clonage de serv1 (cas du TP), elle est déjà présente et cette étape est inutile.**

### 3.5 Sauvegarde de la configuration d'origine

```bash
cp -a /etc/corosync/corosync.conf /etc/corosync/corosync.conf.ori
```

On **copie** au lieu de déplacer (le support utilise `mv`) : en cas de problème, le fichier de référence commenté reste disponible.

### 3.6 Fichier `/etc/corosync/corosync.conf`

```ini
totem {
    version: 2
    cluster_name: cluster_web
    transport: knet
    crypto_cipher: aes256
    crypto_hash: sha256
}

logging {
    to_logfile: yes
    logfile: /var/log/corosync/corosync.log
    to_syslog: yes
    timestamp: on
}

quorum {
    provider: corosync_votequorum
    two_node: 1
}

nodelist {
    node {
        name: serv1
        nodeid: 1
        ring0_addr: 172.16.0.10
    }
    node {
        name: serv2
        nodeid: 2
        ring0_addr: 172.16.0.11
    }
}
```

**À adapter** : `cluster_name` (libre), les `name` (= noms d'hôte réels), les `nodeid` (uniques), les `ring0_addr` (IP réelles).

`two_node: 1` active automatiquement `wait_for_all` : le cluster attend que les deux nœuds se soient vus une première fois avant d'accorder le quorum, puis un seul nœud suffit. C'est ce qui remplace proprement l'ancien `no-quorum-policy=ignore`.

*(Détail des directives supprimées ou corrigées par rapport au support 2016 : voir §9.)*

### 3.7 Vérification syntaxique avant démarrage

```bash
corosync -t
```

Teste le fichier de configuration et sort sans démarrer le démon. À faire systématiquement avant un `systemctl restart`.

### 3.8 Démarrage des services

```bash
systemctl enable --now corosync pacemaker
systemctl status corosync pacemaker --no-pager
```

### 3.9 Clonage de la VM et préparation de serv2

Sur serv1, éteindre proprement puis cloner la VM :

```bash
halt -p
```

Le clonage recopie `authkey` **et** `corosync.conf` : rien à propager à la main.

Sur le clone uniquement :

```bash
hostnamectl set-hostname serv2
rm -f /etc/machine-id && systemd-machine-id-setup
rm -f /etc/ssh/ssh_host_* && dpkg-reconfigure openssh-server
```

> ⚠️ **À exécuter uniquement sur le clone, jamais sur serv1.** Deux VM clonées partagent le même `machine-id` et les mêmes clés SSH : cela casse le DHCP, journald et l'authentification SSH. Point totalement absent du support de 2016.

Puis adapter l'adressage et l'identité :

```bash
nano /etc/network/interfaces     # adresse -> 172.16.0.11
nano /etc/hostname               # serv2
nano /etc/hosts                  # serv1 -> serv2
reboot
```

Sur une Debian sans `ifupdown`, la configuration réseau se fait dans `/etc/systemd/network/` ou via `nmcli`.

Si le clone a conservé un état de cluster incohérent :

```bash
systemctl stop pacemaker corosync
rm -f /var/lib/pacemaker/cib/cib*
systemctl start corosync pacemaker
```

> ⚠️ **Destructif** : efface la configuration locale du cluster. À ne faire que sur un nœud qui n'a pas encore rejoint le cluster ; la CIB sera resynchronisée depuis l'autre nœud.

### 3.10 Vérification du cluster sur les deux nœuds

```bash
corosync-cfgtool -s      # état des liens : LINK ID 0 ... status: connected
corosync-cfgtool -n      # nœuds vus par la couche cluster
corosync-quorumtool -s   # Quorate: Yes, Flags: 2Node WaitForAll
crm status               # vue Pacemaker : nœuds, DC, ressources
```

La sortie de Corosync 3 diffère de celle du support (plus de « RING ID / ring 0 active with no faults »).
`crm_mon` donne la même vue en temps réel (sortie par `Ctrl+C`), `crm_mon -1` en une passe.

### 3.11 Désactivation de STONITH

STONITH (*Shoot The Other Node In The Head*) éteint proprement un nœud défaillant — utile surtout avec des disques partagés. Sans agent de fencing déclaré, Pacemaker refuse de démarrer les ressources et `crm_verify` remonte des erreurs.

```bash
crm configure property stonith-enabled=false
crm_verify -L -V
```

> ⚠️ **En production**, on configure un vrai agent (`fence_virsh`, `fence_vmware_soap`, `fence_ipmilan` ou SBD) : sans fencing, un split-brain corrompt les données partagées.

### 3.12 Quorum sur deux nœuds

```bash
crm configure property no-quorum-policy=stop
```

Valeur par défaut, à laisser telle quelle : avec `two_node: 1` dans `corosync.conf`, le quorum est déjà géré correctement.

> **Correction 2016** : `no-quorum-policy="ignore"` reste une valeur acceptée en Pacemaker 2.x/3.x, mais elle est déconseillée ici. Elle désactive globalement la politique de quorum, alors que `two_node: 1` traite précisément le cas des deux nœuds (quorum fixé artificiellement à 1 + `wait_for_all`, qui évite la course au démarrage où chaque nœud se croirait seul). On garde donc `stop`.

### 3.13 Consultation de la configuration générée

```bash
crm configure show        # CIB en syntaxe crmsh, lisible
cibadmin -Q | head -n 40  # CIB brute au format XML
```

> ⚠️ Ne **jamais** éditer `/var/lib/pacemaker/cib/cib.xml` à la main — utiliser `crm configure edit`.

---

## 4. Activité 2 — Ressources IPFailover et serviceWeb

Le client ne doit connaître qu'une seule adresse IP, rattachée au cluster et non à une machine précise.

### 4.1 Création de la ressource d'adresse IP virtuelle

```bash
crm configure primitive IPFailover ocf:heartbeat:IPaddr2 \
  params ip=172.16.0.12 cidr_netmask=24 nic=ens33 iflabel=VIP \
  op monitor interval=10s timeout=20s \
  op start interval=0s timeout=30s \
  op stop interval=0s timeout=40s
```

| Paramètre | À adapter ? | Détail |
|---|---|---|
| `ip=172.16.0.12` | ✅ oui | adresse IP virtuelle du cluster |
| `cidr_netmask=24` | ✅ oui | masque de sous-réseau |
| `nic=ens33` | ✅ oui | nom réel de l'interface relevé en §2.3 |
| `iflabel=VIP` | libre | alias visible d'interface |
| `interval` / `timeout` | optionnel | test de vie 10 s, démarrage 30 s, arrêt 40 s |

`ocf` = classe, `heartbeat` = fournisseur, `IPaddr2` = script appelé.

> ⚠️ Le label produit l'alias `<interface>:VIP`, soumis à la limite noyau **IFNAMSIZ de 15 caractères**. `ens33:VIP` (9) ou `enp0s31f6:VIP` (13) passent ; une interface longue plus un label long font échouer le démarrage avec une erreur peu explicite. En cas de doute, raccourcir le label ou retirer `iflabel` (la VIP reste visible avec `ip addr show`).

```bash
crm ra info ocf:heartbeat:IPaddr2
```

Liste tous les paramètres et timeouts par défaut de l'agent — à consulter avant d'inventer des valeurs.

### 4.2 Vérifier où la ressource a démarré

```bash
crm status
ip -c addr show dev ens33
```

Pacemaker place la ressource sur un nœud arbitrairement tant qu'aucune contrainte n'existe. La VIP apparaît comme adresse `secondary` sur l'interface.

> **Correction 2016** : `ifconfig` n'est plus installé par défaut (`net-tools`) — utiliser `ip addr`.

### 4.3 Définir une préférence de nœud

```bash
crm resource move IPFailover serv1
```

Migre la ressource et fait écrire par Pacemaker une contrainte de localisation `cli-prefer-IPFailover`.

```bash
crm resource clear IPFailover
```

Supprime cette contrainte une fois la migration constatée.

> **Correction 2016** : `clear` est le nom actuel en crmsh 4 ; `unmove` reste accepté comme alias. Ce qui compte, c'est de ne pas oublier l'étape : sans elle, la contrainte reste permanente et le comportement du cluster devient difficile à expliquer en soutenance.

Contrainte de localisation explicite, si besoin :

```bash
crm configure location <nom_contrainte> <id_ressource> <score>: <noeud>
```

### 4.4 Empêcher le retour automatique (auto failback)

```bash
crm configure rsc_defaults resource-stickiness=100
```

La ressource reste sur le nœud où elle tourne même si le nœud préféré revient en ligne ; le retour devient manuel.

> **Correction 2016** : `crm configure property default-resource-stickiness=100` n'existe plus en Pacemaker 2 — c'est `rsc_defaults resource-stickiness`.

### 4.5 Tester l'accès par la VIP

```bash
curl -I http://172.16.0.12/applifrais/
```

Valide la Q4 : l'application répond bien sur l'adresse virtuelle.

### 4.6 Service impacté par le changement d'adresse (Q5)

```bash
grep -R "ServerName" /etc/apache2/sites-enabled/
```

C'est le **DNS** qui est impacté : l'enregistrement A du FQDN de l'application doit pointer sur `172.16.0.12`, et non plus sur l'IP réelle d'un nœud. Corollaire côté Apache : le `ServerName` du vhost, et surtout le **certificat TLS** (CN/SAN), doivent correspondre à ce FQDN, sinon HTTPS casse à la bascule.

### 4.7 Ressource du service Web

```bash
crm configure primitive serviceWeb systemd:apache2 \
  op monitor interval=30s timeout=30s \
  op start interval=0s timeout=60s \
  op stop interval=0s timeout=60s
```

> **Correction majeure** : le support utilise `lsb:apache2`. La classe LSB s'appuie sur `/etc/init.d/apache2`, qui n'est plus qu'un wrapper systemd sur Debian moderne : le monitoring est peu fiable. On utilise la classe `systemd:`.

**Variante plus fine, avec supervision HTTP réelle :**

```bash
a2enmod status
crm configure primitive serviceWeb ocf:heartbeat:apache \
  params configfile=/etc/apache2/apache2.conf \
         statusurl="http://127.0.0.1/server-status" \
  op monitor interval=30s timeout=30s \
  op start interval=0s timeout=60s \
  op stop interval=0s timeout=60s
```

L'agent OCF interroge `/server-status` : il détecte un Apache « lancé mais qui ne répond plus », ce que `systemd:` ne voit pas. Nécessite `mod_status` activé et accessible depuis `127.0.0.1`.

> 🔸 **Au cas où** — erreur sur la primitive, la refaire proprement :
> ```bash
> crm resource stop serviceWeb
> crm configure delete serviceWeb
> ```

### 4.8 Vérifier que le service ne tourne que sur un nœud (Q8)

```bash
ss -lntp | grep ':80'
```

Sur le nœud actif, Apache écoute ; sur le passif, aucune ligne.

> **Correction 2016** : `netstat` est remplacé par `ss`. `nmap localhost -p 80` reste valable (`open` = actif, `closed` = passif) mais `ss` suffit et est natif.

### 4.9 Grouper les ressources (Q9)

```bash
crm configure group servweb IPFailover serviceWeb meta migration-threshold=5
```

Par défaut Pacemaker répartit les ressources entre nœuds : la VIP et Apache se retrouvent séparés. Le groupe force la colocation **et** un ordre de démarrage (VIP puis Apache, arrêt en sens inverse). `migration-threshold=5` : au bout de 5 échecs, le groupe migre définitivement.

```bash
crm configure rsc_defaults failure-timeout=600s
```

Complément utile : les compteurs d'échec s'effacent seuls au bout de 10 minutes, ce qui évite un `cleanup` manuel après chaque test.

> ⚠️ `failure-timeout` est un **méta-attribut de ressource**, pas une propriété de cluster. Passé en `crm configure property`, Pacemaker l'enregistre sans broncher mais ne l'applique jamais — panne silencieuse difficile à diagnostiquer. Il se met dans `rsc_defaults`, ou directement sur la ressource avec `meta failure-timeout=600s`.

### 4.10 Repère visuel du serveur actif

Pour identifier à l'œil quel nœud sert l'application, modifier l'en-tête avec un texte différent sur chaque serveur :

```bash
nano /var/www/applifrais/include/_entete.inc.html
```

---

## 5. Activité 3 — Réplication MariaDB maître-esclave

### 5.1 Principe

- Le **maître** journalise toutes les écritures (`INSERT`, `UPDATE`, `DELETE`) dans un log binaire.
- L'**esclave** se connecte au maître, lit ce journal depuis une position donnée et rejoue les requêtes localement.
- Les bases doivent être **identiques** sur les deux serveurs avant de démarrer la réplication.

### 5.2 Installation et point de vocabulaire

```bash
apt install -y mariadb-server
systemctl status mariadb --no-pager
```

Sur Debian, le paquet est `mariadb-server` et l'unité `mariadb.service` (`mysql.service` n'est qu'un alias). Les binaires clients sont `mariadb`, `mariadb-dump`, `mariadb-admin` — `mysql` et `mysqldump` ne sont plus que des liens de compatibilité.

Préparer le répertoire de logs sur **les deux nœuds** :

```bash
mkdir -p /var/log/mysql
chown -R mysql:mysql /var/log/mysql
chmod -R 750 /var/log/mysql
```

### 5.3 `/etc/mysql/mariadb.conf.d/50-server.cnf` — nœud maître (serv1)

```ini
[mysqld]
bind-address               = 0.0.0.0
log_error                  = /var/log/mysql/error.log
server_id                  = 1
log_bin                    = /var/log/mysql/mysql-bin.log
binlog_format              = ROW
binlog_expire_logs_seconds = 864000
max_binlog_size            = 100M
binlog_do_db               = gsb_valide
```

`server_id` doit être unique dans la chaîne de réplication. `log_bin` active le journal binaire lu par l'esclave. `binlog_do_db` restreint la réplication à la base voulue.

> **Corrections 2016** : `expire_logs_days = 10` est déprécié → `binlog_expire_logs_seconds = 864000` (= 10 jours). On définit `bind-address = 0.0.0.0` au lieu de commenter la ligne.
>
> `binlog_format = ROW` est ajouté, et ce n'est **pas cosmétique** : en *statement-based*, `binlog_do_db` et `replicate_do_db` filtrent sur la base par défaut (celle du dernier `USE`) et non sur la table réellement écrite. Une requête `INSERT INTO autrebase.table` lancée depuis `USE gsb_valide` serait donc répliquée à tort, et l'inverse serait ignoré. En `ROW`, le filtrage porte sur la table réelle.

### 5.4 Même fichier — nœud esclave (serv2)

```ini
[mysqld]
bind-address       = 0.0.0.0
log_error          = /var/log/mysql/error.log
server_id          = 2
log_bin            = /var/log/mysql/mysql-bin.log
relay_log          = /var/log/mysql/mysql-relay-bin.log
log_slave_updates  = ON
replicate_do_db    = gsb_valide
master_retry_count = 100000
```

`log_slave_updates` est ajouté dès maintenant : il sera indispensable en activité 4.

> **Correction du support 2016** : `master-retry-count` reste une option serveur valide et se met bien dans le `.cnf`. C'est sa **description** qui est fausse dans le support : ce n'est pas « se reconnecter toutes les 20 secondes », c'est le **nombre de tentatives** de connexion avant abandon définitif. L'intervalle entre deux tentatives est fixé par `MASTER_CONNECT_RETRY` (60 s par défaut).
>
> ⚠️ Ne pas reprendre la valeur `20` du support : l'esclave abandonnerait après 20 essais, soit une vingtaine de minutes d'indisponibilité du maître. Les valeurs par défaut sont 86400 jusqu'à MariaDB 10.5 et 100000 à partir de 10.6 — on les garde.

### 5.5 Redémarrage

```bash
systemctl restart mariadb
journalctl -u mariadb -n 30 --no-pager
```

Toujours relire les logs après redémarrage : une directive inconnue empêche MariaDB de démarrer (`unknown variable`).

> 🔸 **Au cas où** — MariaDB ne redémarre pas :
> ```bash
> tail -f /var/log/mysql/error.log
> journalctl -xeu mariadb.service
> ```

### 5.6 Création du compte de réplication (sur le maître)

```bash
mariadb -u root -p     # ou simplement : sudo mariadb (auth unix_socket)
```

```sql
CREATE USER 'replicateur'@'172.16.0.11' IDENTIFIED BY 'Btssio2017';
GRANT REPLICATION SLAVE ON *.* TO 'replicateur'@'172.16.0.11';
FLUSH PRIVILEGES;
```

> **Correction 2016** : MariaDB accepte encore `GRANT … IDENTIFIED BY '<mdp>'`, mais MySQL 8 l'a supprimée. La forme `CREATE USER` puis `GRANT` fonctionne sur les deux : c'est celle à retenir. Le privilège `REPLICATION SLAVE` suffit, ne pas donner `ALL`.
>
> Le TP utilise `'replicateur'@'%'` : restreindre à l'IP du nœud pair est plus propre.

### 5.7 Synchronisation initiale des données

**Méthode recommandée aujourd'hui**, sans verrou global prolongé :

```bash
mariadb-dump --single-transaction --master-data=2 --databases gsb_valide > /root/gsb_valide.sql
```

`--single-transaction` fige une vue cohérente (InnoDB) sans bloquer les écritures ; `--master-data=2` ajoute en tête du dump, en commentaire, le `CHANGE MASTER TO` avec le fichier et la position exacts.

> ⚠️ `--source-data` est le nom MySQL 8.0.26+ de cette option. MariaDB ne le connaît pas et sort `unknown option`. Sur MariaDB, c'est bien `--master-data`.
>
> **Corrections 2016** : `mysqldump` → `mariadb-dump` ; ne jamais passer le mot de passe en clair via `-p<motDePasse>` (visible dans `ps`), utiliser `-p` seul ou `--defaults-extra-file`.

**Méthode « historique » du support**, si l'on veut respecter le TP à la lettre — sur le maître :

```sql
FLUSH TABLES WITH READ LOCK;
SHOW BINLOG STATUS;
```

Bloque toutes les écritures et affiche `File` et `Position` à noter (ils changent à chaque fois). Garder la session ouverte jusqu'au `UNLOCK TABLES`.

`SHOW MASTER STATUS` reste accepté ; `SHOW BINLOG STATUS` est la forme actuelle côté MariaDB depuis 10.5.2 (`SHOW BINARY LOG STATUS` côté MySQL 8.2+).

> ⚠️ Tant que `UNLOCK TABLES;` n'est pas saisi, l'application est en lecture seule — ne pas laisser la session ouverte.

Restauration sur l'esclave :

```bash
mariadb < /root/gsb_valide.sql
```

Les bases doivent être strictement identiques avant de lancer la réplication, sinon erreur `Duplicate entry … Error_code: 1062`.

### 5.8 Configuration de l'esclave (serv2)

```sql
STOP SLAVE;
CHANGE MASTER TO
  MASTER_HOST='172.16.0.10',
  MASTER_USER='replicateur',
  MASTER_PASSWORD='Btssio2017',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=328,
  MASTER_CONNECT_RETRY=20;
START SLAVE;
```

`MASTER_LOG_FILE` et `MASTER_LOG_POS` sont les valeurs relevées à l'étape précédente. `MASTER_CONNECT_RETRY=20` fixe le **délai en secondes** entre deux tentatives de reconnexion.

> ⚠️ **Ne pas ajouter `MASTER_RETRY_COUNT` ici.** Cette option de `CHANGE MASTER TO` n'existe qu'à partir de MariaDB 12.0.1 (et côté MySQL). Sur Debian 12 (MariaDB 10.11) et Debian 13 (11.8), elle provoque une erreur de syntaxe qui fait échouer toute la commande. Le nombre de tentatives se règle dans le `.cnf` (§5.4).

**Variante GTID** (recommandée en MariaDB 10.x/11.x, plus robuste) :

```sql
STOP SLAVE;
CHANGE MASTER TO
  MASTER_HOST='172.16.0.10',
  MASTER_USER='replicateur',
  MASTER_PASSWORD='Btssio2017',
  MASTER_USE_GTID=slave_pos;
START SLAVE;
```

Plus besoin de relever fichier/position à la main : le GTID suit tout seul le point de reprise, y compris après une bascule. C'est l'approche moderne, absente du support de 2016.

Puis, sur le maître, si l'on avait verrouillé :

```sql
UNLOCK TABLES;
```

### 5.9 Contrôle de l'état de la réplication

```sql
SHOW SLAVE STATUS\G
```

Trois champs à vérifier : `Slave_IO_Running: Yes`, `Slave_SQL_Running: Yes`, `Seconds_Behind_Master: 0`. En cas d'erreur, lire `Last_IO_Error` et `Last_SQL_Error`.

> **Note de syntaxe** : MariaDB 10.5+ accepte les alias `SHOW REPLICA STATUS`, `START REPLICA`, `STOP REPLICA`. Sous MySQL 8.0.23+, ces formes sont **obligatoires** : `CHANGE REPLICATION SOURCE TO SOURCE_HOST=…, SOURCE_USER=…, SOURCE_LOG_FILE=…, SOURCE_LOG_POS=…`.

---

## 6. Activité 4 — Multi-maître et intégration au cluster

### 6.1 Problématique

Avec une réplication simple maître-esclave, si le maître tombe puis revient, Pacemaker lui redonne la priorité — mais il n'a pas connaissance des données écrites sur l'esclave entre-temps. **Risque de perte de données.**

**Solution** : chaque serveur est à la fois maître et esclave de l'autre (deux réplications croisées). Les données restent à jour sur les deux nœuds, quel que soit celui qui est actif.

### 6.2 Compléter les deux fichiers de configuration

Ajouter sur chaque nœud ce qui lui manque pour jouer les deux rôles :

```ini
[mysqld]
server_id                = <1 sur serv1, 2 sur serv2>
log_bin                  = /var/log/mysql/mysql-bin.log
relay_log                = /var/log/mysql/mysql-relay-bin.log
log_slave_updates        = ON
binlog_format            = ROW
binlog_do_db             = gsb_valide
replicate_do_db          = gsb_valide
auto_increment_increment = 2
auto_increment_offset    = <1 sur serv1, 2 sur serv2>
```

`log_slave_updates` est **obligatoire** : sans lui, un nœud ne journalise pas dans son binlog les écritures reçues par réplication, et il ne peut pas détecter qu'un événement qu'il a déjà exécuté lui revient (boucle infinie).

`auto_increment_increment` = nombre total de maîtres (2 ici) ; `auto_increment_offset` = valeur unique par maître. Évite les collisions de clés auto-incrémentées.

```bash
systemctl restart mariadb     # sur les deux nœuds
```

### 6.3 Créer le compte de réplication dans l'autre sens

Sur **serv2**, qui devient à son tour maître de serv1 :

```sql
CREATE USER 'replicateur'@'172.16.0.10' IDENTIFIED BY 'Btssio2017';
GRANT REPLICATION SLAVE ON *.* TO 'replicateur'@'172.16.0.10';
FLUSH PRIVILEGES;
SHOW BINLOG STATUS;
```

Relever `File` et `Position`.

### 6.4 Transformer serv1 en esclave de serv2

Sur **serv1** :

```sql
STOP SLAVE;
CHANGE MASTER TO
  MASTER_HOST='172.16.0.11',
  MASTER_USER='replicateur',
  MASTER_PASSWORD='Btssio2017',
  MASTER_USE_GTID=slave_pos;
START SLAVE;
SHOW SLAVE STATUS\G
```

*(Variante fichier/position : `MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=328` à la place de `MASTER_USE_GTID`.)*

On obtient deux réplications maître-esclave croisées : chaque nœud peut reprendre l'activité avec des données à jour, sans procédure manuelle de bascule.

**Test dans les deux sens** — sur serv1 :

```sql
USE gsb_valide;
UPDATE Visiteur SET mdp='titi' WHERE login='agest';
```

Sur serv2 :

```sql
USE gsb_valide;
SELECT * FROM Visiteur;
SHOW SLAVE STATUS\G
```

Puis refaire un `UPDATE` depuis serv2 et vérifier avec un `SELECT` sur serv1.

> 🔸 **Au cas où** — réplication arrêtée sur un serveur :
> ```sql
> STOP SLAVE;
> START SLAVE;
> SHOW SLAVE STATUS\G    -- lire Last_Error si Slave_IO_Running / Slave_SQL_Running = No
> ```

### 6.5 Ressource MariaDB en mode actif/actif

**Solution simple et fiable pour le TP :**

```bash
crm configure primitive serviceMySQL systemd:mariadb \
  op monitor interval=30s timeout=30s \
  op start interval=0s timeout=120s \
  op stop interval=0s timeout=120s
crm configure clone cServiceMySQL serviceMySQL meta interleave=true
```

Le clone de type *anonymous* démarre la ressource sur les deux nœuds : il n'y a pas de bascule de MariaDB, chaque nœud a déjà sa base à jour. Pacemaker ne désactive MariaDB sur aucun des deux nœuds ; c'est la réplication multi-maître qui assure la cohérence.

**Variante avec l'agent OCF** (plus proche du support, plus capricieuse) :

```bash
crm configure primitive serviceMySQL ocf:heartbeat:mysql \
  params binary=/usr/sbin/mariadbd \
         config=/etc/mysql/mariadb.cnf \
         datadir=/var/lib/mysql \
         pid=/run/mysqld/mysqld.pid \
         socket=/run/mysqld/mysqld.sock \
  op monitor interval=30s timeout=30s \
  op start interval=0s timeout=120s \
  op stop interval=0s timeout=120s
```

> **Corrections 2016** : le support écrit `ocf:Corosync:mysql` — le fournisseur est **`heartbeat`**, pas `Corosync`. Le binaire est `mariadbd` depuis MariaDB 10.5 (et non `mysqld`). Le socket canonique est `/run/mysqld/mysqld.sock` (`/var/run` n'est qu'un lien symbolique).

> 🔸 **Au cas où** — mention « Unpromoted » dans `crm status`, recréer le clone :
> ```bash
> crm resource stop cServiceMySQL
> crm configure delete cServiceMySQL
> crm configure clone cServiceMySQL serviceMySQL meta promotable=false clone-max=2 clone-node-max=1 interleave=true
> crm status
> ```
> 🔸 Erreur sur la primitive : `crm resource cleanup serviceMySQL`

### 6.6 Ordonner le démarrage

```bash
crm configure order o-mysql-avant-web Mandatory: cServiceMySQL servweb
```

Garantit que MariaDB est opérationnel avant qu'Apache ne démarre sur le nœud actif — sinon l'application affiche une erreur de connexion à la base pendant quelques secondes après chaque bascule.

### 6.7 Contrôle final

```bash
crm configure show
crm status
```

Résultat attendu : le groupe `servweb` sur un seul nœud, le `Clone Set: cServiceMySQL` démarré sur les deux.


---

## 7. Tests de validation

### 7.1 Mise en maintenance d'un nœud

```bash
crm node standby serv1
crm status
```

Le nœud n'héberge plus de ressources : le groupe bascule sur l'autre. **Bascule propre et immédiate**, les services sont arrêtés puis redémarrés dans l'ordre.

```bash
crm node online serv1
```

Remet le nœud en service. Avec `resource-stickiness=100`, les ressources **ne reviennent pas** automatiquement.

### 7.2 Extinction brutale du nœud actif

```bash
systemctl poweroff      # ou : halt -p
```

> ⚠️ Test destructif pour les sessions en cours. Différence avec le `standby` : Corosync doit d'abord détecter la perte du nœud (délai de token), le nœud passe `OFFLINE`, la bascule est donc **plus lente** et les connexions en cours sont coupées.

Vérification côté service :

```bash
ss -lntp | grep ':80'     # ou : nmap localhost -p 80
```

### 7.3 Panne du seul service Web

```bash
systemctl mask apache2
systemctl stop apache2
```

> ⚠️ Empêche volontairement tout redémarrage d'Apache, ce qui force Pacemaker à échouer puis à migrer le groupe après `migration-threshold` échecs. Sans cela, Pacemaker relance le service et aucune bascule n'a lieu — **c'est le piège du TP**.

Retour arrière **obligatoire** après le test :

```bash
systemctl unmask apache2
crm resource cleanup servweb
```

Sans quoi le nœud reste inéligible pour héberger les ressources.

### 7.4 Validation de la réplication

```sql
SELECT COUNT(*) FROM gsb_valide.Visiteur;
```

À exécuter sur les deux nœuds après saisie d'une fiche de frais depuis l'application : les compteurs doivent être identiques.

### 7.5 Tableau de recette (Q3 act.3 / Q4 act.4)

| Action à effectuer | Résultat attendu | Résultat obtenu | Statut |
|---|---|---|---|
| Ressources sur serv1, saisie d'une fiche de frais | Fiche visible sur les deux bases | | |
| `crm node standby serv1` | Groupe `servweb` démarré sur serv2, VIP répond | | |
| Saisie d'une seconde fiche via la VIP | Écriture acceptée sur serv2 | | |
| `crm node online serv1` | serv1 en ligne, réplication `Yes`/`Yes` dans les deux sens | | |
| Comparaison des deux bases | Nombre de fiches identique des deux côtés | | |

---

## 8. Dépannage

### 8.1 Logs du cluster

```bash
journalctl -u corosync -u pacemaker -f
```

Flux temps réel des deux démons. Complément : `/var/log/corosync/corosync.log` et `/var/log/pacemaker/pacemaker.log`.

```bash
crm_mon -rfn1
```

Vue détaillée en une passe : ressources inactives (`-r`), compteurs d'échec (`-f`), regroupées par nœud (`-n`). **La commande la plus utile** pour comprendre pourquoi une ressource ne démarre pas.

### 8.2 Compteurs d'échec

```bash
crm resource failcount <id_ressource> show <noeud>
crm resource cleanup <id_ressource>
```

Après un certain nombre d'erreurs, le nœud fautif ne peut plus héberger la ressource tant que les compteurs ne sont pas effacés.

### 8.3 Sauvegarde et restauration de la configuration

```bash
crm configure save /root/sauveha.conf
crm configure save xml /root/sauveha.xml
```

Export de la CIB. À faire avant toute manipulation risquée.

```bash
crm configure load replace /root/sauveha.conf
```

> ⚠️ Remplace **intégralement** la configuration courante. Pour un ajout, utiliser `load update`.

### 8.4 Erreur de réplication `Duplicate entry … 1062`

**Cause** : les deux bases n'étaient pas identiques au départ, ou une écriture a été faite par erreur sur l'esclave.

```sql
SHOW SLAVE STATUS\G
```

Lire `Last_SQL_Error` pour identifier la table et la position en cause.

**Solution propre** : redump du maître vers l'esclave.

```bash
# sur l'esclave
mariadb -e "STOP SLAVE;"
mariadb-dump -h 172.16.0.10 -u root -p --single-transaction --databases gsb_valide --add-drop-table > /root/sauvBases.sql
mariadb < /root/sauvBases.sql
```

Puis relever `SHOW BINLOG STATUS;` sur le maître et refaire le `CHANGE MASTER TO` + `START SLAVE;`.

**Contournement rapide (labo uniquement)** :

```sql
STOP SLAVE;
SET GLOBAL sql_slave_skip_counter = 1;
START SLAVE;
```

> ⚠️ Dangereux : saute l'événement en erreur et crée une divergence silencieuse entre les bases. Acceptable en labo pour débloquer, **jamais en production** sans analyse.
>
> ⚠️ Ce compteur saute un événement du log binaire, ce qui n'a de sens qu'en réplication **par fichier/position**. Si l'esclave a été configuré avec `MASTER_USE_GTID=slave_pos` (§5.8), c'est le mauvais outil : il faut faire avancer la position GTID.

```sql
STOP SLAVE;
SET GLOBAL gtid_slave_pos = '<domaine>-<serveur>-<sequence>';
CHANGE MASTER TO MASTER_USE_GTID=slave_pos;
START SLAVE;
```

Encore plus dangereux que le compteur : on déclare explicitement à l'esclave qu'il a traité une transaction qu'il n'a pas exécutée. La position courante se lit avec `SELECT @@global.gtid_slave_pos;` et celle du maître avec `SELECT @@global.gtid_binlog_pos;`.

### 8.5 Décalage File / Position entre maître et esclave

```sql
SHOW BINLOG STATUS;      -- sur le maître
SHOW SLAVE STATUS\G      -- sur l'esclave : comparer Master_Log_File et Read_Master_Log_Pos
```

Cause fréquente : une écriture faite par erreur sur l'esclave. Solution : resynchroniser les bases (§8.4).

### 8.6 Gestion des journaux binaires

```sql
SHOW BINARY LOGS;
PURGE BINARY LOGS TO 'mysql-bin.000005';
```

Le maître ignore combien d'esclaves le suivent : ses binlogs s'accumulent. `PURGE` supprime tout jusqu'au fichier indiqué, **exclu** — ne jamais purger le fichier en cours.

> **Correction 2016** : `SHOW MASTER LOGS` / `PURGE MASTER LOGS` sont remplacés par `SHOW BINARY LOGS` / `PURGE BINARY LOGS`.
>
> ⚠️ Ne jamais supprimer ces fichiers avec `rm` : l'index `mysql-bin.index` se désynchronise et `PURGE` cesse de fonctionner.

---

## 9. Tableau des corrections 2016 → aujourd'hui

| Support CERTA (2016) | Version actuelle | Motif |
|---|---|---|
| `service corosync status`, `/etc/init.d/corosync status` | `systemctl status corosync` | SysV remplacé par systemd |
| `two_nodes: 1` | `two_node: 1` | directive inexistante, ignorée en silence |
| `expected_votes: 2` | *(supprimé)* | déduit du `nodelist` |
| `clear_node_high_bit: yes` | *(supprimé)* | obsolète avec `nodeid` explicites |
| `service { ver: 0 name: pacemaker }` | *(supprimé)* + `systemctl enable pacemaker` | Pacemaker 2 n'est plus démarré par Corosync |
| `crypto_hash: sha1` | `crypto_hash: sha256` | SHA-1 obsolète |
| multicast implicite | `transport: knet` | défaut Corosync 3 |
| `no-quorum-policy=ignore` | `two_node: 1` + `no-quorum-policy=stop` | `ignore` toujours valide mais déconseillé |
| `default-resource-stickiness=100` | `rsc_defaults resource-stickiness=100` | propriété supprimée en Pacemaker 2.0.0 |
| `crm resource unmove` | `crm resource clear` | `clear` est le nom actuel, `unmove` reste un alias |
| `lsb:apache2` | `systemd:apache2` ou `ocf:heartbeat:apache` | classe LSB non fiable sous systemd |
| `ocf:Corosync:mysql` | `ocf:heartbeat:mysql` ou `systemd:mariadb` | fournisseur erroné dans le support |
| `binary=mysqld` | `binary=/usr/sbin/mariadbd` | binaire renommé (MariaDB 10.5+) |
| `/var/run/mysqld/mysqld.sock` | `/run/mysqld/mysqld.sock` | `/var/run` n'est qu'un lien |
| `nic=eth0` | `nic=ens33` / `ens18` / `enp0s3` | noms d'interfaces prédictibles |
| `ifconfig` | `ip -c addr show` | `net-tools` non installé |
| `netstat` | `ss -lntp` | idem |
| `corosync-keygen` + frappe clavier | `corosync-keygen` (instantané) | entropie lue sur `/dev/urandom` |
| `mysqldump`, `mysql` | `mariadb-dump`, `mariadb` | binaires renommés |
| `expire_logs_days = 10` | `binlog_expire_logs_seconds = 864000` | directive dépréciée |
| `master-retry-count = 20` | `master_retry_count = 100000` (dans le `.cnf`) | l'option est valide : c'est sa description qui était fausse |
| `GRANT … IDENTIFIED BY` | `CREATE USER` puis `GRANT` | supprimé en MySQL 8 |
| `#bind-address = 127.0.0.1` | `bind-address = 0.0.0.0` | plus explicite et reproductible |
| `SHOW MASTER STATUS` | `SHOW BINLOG STATUS` (MariaDB 10.5.2+) | renommage, ancien nom conservé comme alias |
| `SHOW MASTER LOGS` / `PURGE MASTER LOGS` | `SHOW BINARY LOGS` / `PURGE BINARY LOGS` | renommage |
| `STOP/START SLAVE`, `SHOW SLAVE STATUS` | alias `REPLICA` (MariaDB 10.5+), obligatoire en MySQL 8.0.23+ | terminologie révisée |
| position binlog relevée à la main | `MASTER_USE_GTID=slave_pos` | GTID, reprise automatique |
| *(absent)* | `systemctl disable --now apache2 mariadb` | sinon systemd et Pacemaker se disputent les services |
| *(absent)* | `crm configure rsc_defaults failure-timeout=600s` | `failure-timeout` est un méta-attribut, ignoré en `property` |
| *(absent)* | `binlog_format = ROW` | `binlog_do_db` / `replicate_do_db` filtrent mal en statement-based |
| *(absent)* | régénération `/etc/machine-id` + clés SSH après clonage | conflits d'identité entre VM clonées |
| `stonith-enabled=false` | idem en labo, agent réel en production | sans fencing, risque de split-brain |

---

## 10. Aide-mémoire des commandes

### Cluster

```bash
crm status                                   # état du cluster, une passe
crm_mon -rfn                                 # temps réel, détaillé (Ctrl+C pour sortir)
crm_mon -1                                   # une passe, équivalent de crm status
corosync-cfgtool -s                          # état des liens
corosync-quorumtool -s                       # état du quorum
crm configure show                           # configuration lisible
crm configure edit [<id_ressource>]          # édition dans vi, appliquée à la sauvegarde
crm configure verify                         # validation de la configuration
crm_verify -L -V                             # validation, sortie verbeuse
crm resource start|stop <id_ressource>       # démarrage / arrêt d'une ressource
crm resource move <id_ressource> <noeud>     # migration (crée une contrainte)
crm resource clear <id_ressource>            # suppression de la contrainte (alias : unmove)
crm resource cleanup <id_ressource>          # effacement des compteurs d'erreur
crm configure delete <id_ressource>          # suppression (stopper la ressource avant)
crm node standby|online <noeud>              # entrée / sortie de maintenance
crm ra list ocf heartbeat                    # agents OCF disponibles
crm ra info ocf:heartbeat:<agent>            # paramètres et timeouts par défaut
crm configure clone <nom_clone> <id_ressource>     # ressource en actif/actif
crm configure rsc_defaults <option>=<valeur>       # méta-attribut par défaut
crm configure save /root/sauveha.conf              # sauvegarde de la configuration
crm configure load update /root/sauveha.conf       # application incrémentale
```

### MariaDB

```sql
SHOW BINLOG STATUS;                          -- fichier et position courants (maître)
SHOW SLAVE STATUS\G                          -- état complet de l'esclave
STOP SLAVE; START SLAVE;                     -- arrêt / relance du thread de réplication
SELECT @@global.gtid_slave_pos;              -- position GTID de l'esclave
SELECT @@global.gtid_binlog_pos;             -- position GTID du maître
SHOW BINARY LOGS;                            -- liste des journaux binaires
PURGE BINARY LOGS TO '<fichier_binlog>';     -- purge jusqu'au fichier indiqué (exclu)
FLUSH TABLES WITH READ LOCK; UNLOCK TABLES;  -- verrou global (méthode historique)
```

---

## 11. Sources de vérification

Les corrections apportées au support CERTA 2016 ont été vérifiées dans :

- **Corosync 3** : manpages `votequorum(5)` et `corosync(8)`, Debian bookworm — comportement de `two_node` / `wait_for_all`, option `-t`
- **Pacemaker 2.1** : *Pacemaker Explained*, chapitre « Resource Operations » (`failure-timeout` comme méta-attribut) et `pacemaker-schedulerd(7)` (valeurs acceptées de `no-quorum-policy`)
- **Pacemaker 2.0.0 release notes** — suppression des alias `default-resource-stickiness`, `is-managed-default`, `default-action-timeout`
- **crmsh** : *Quick Comparison of pcs and crm shell*, documentation ClusterLabs 2.1
- **MariaDB KB** : `CHANGE MASTER TO` (disponibilité de `MASTER_RETRY_COUNT` à partir de 12.0.1), `SHOW BINLOG STATUS` (10.5.2), options `mariadbd` (`--master-retry-count`), options `mariadb-dump` (`--master-data`)
- **MySQL 8.0 Reference Manual** : `mysqldump` (`--source-data`, 8.0.26+)
