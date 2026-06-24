# Guide Lab : Oracle Data Guard 19c Physical Standby
## Base ELIORA — VMware Workstation / Oracle Linux 8.10

> **Usage** : Référence personnelle / mémo lab  
> **Auteur** : Tony (Antoine Koffi KPOTI)  
> **Date** : Juin 2026  
> **Statut** : ✅ Environnement fonctionnel

---

## Sommaire

1. [Création des VMs VMware Workstation](#1-création-des-vms-vmware-workstation)
2. [Installation d'Oracle Linux 8.10](#2-installation-doracle-linux-810)
3. [Installation d'Oracle Database 19c](#3-installation-doracle-database-19c)
4. [Configuration Oracle Data Guard 19c Physical Standby](#4-configuration-oracle-data-guard-19c-physical-standby)

---

## Environnement de référence

| Paramètre | Valeur |
|---|---|
| Hyperviseur | VMware Workstation |
| OS | Oracle Linux 8.10 (x86_64) |
| Oracle Version | 19c (19.3.0.0.0) |
| Base de données | ELIORA |
| VM Primary | vm1-oratony19 |
| VM Standby | vm2-oratony19 |
| ORACLE_BASE | /opt/oracle |
| ORACLE_HOME | /opt/oracle/product/19.0.0/dbhome_1 |
| ORACLE_SID | ELIORA (identique sur les deux VMs) |

---

## 1. Création des VMs VMware Workstation

Les deux VMs sont identiques en termes de configuration matérielle.

### 1.1 Caractéristiques des VMs

| Ressource | Valeur |
|---|---|
| RAM | 5 Go |
| CPU | 8 vCPU |
| OS ISO | Oracle Linux 8.10 |
| Réseau | Bridged (accès réseau local entre VMs) |

### 1.2 Plan de partitionnement des disques

| Point de montage | Taille | Groupe LVM | Volume logique |
|---|---|---|---|
| /boot | 1,5 Go | — | nvme0n1p1 |
| / | 20 Go | vg | lv_system_root |
| /opt | 30 Go | vg | lv_opt |
| /home | 5 Go | vg | lv_home |
| /tmp | 5 Go | vg | lv_system_tmp |
| /var | 5 Go | vg | lv_system_var |
| /opt/oracle/oradata | 70 Go | vg | lv_oracle_data |
| /opt/oracle/orasave | 30 Go | vg | lv_oracle_save |

> **Note** : Le partitionnement LVM permet de redimensionner les volumes oracle à chaud si besoin.

### 1.3 Création de la VM dans VMware Workstation

```
1. File > New Virtual Machine > Custom
2. Hardware compatibility : Workstation 17.x (ou version installée)
3. Installer disc image file (iso) : sélectionner OL-8-10-0-BaseOS-x86_64.iso
4. Guest OS : Linux > Oracle Linux 8 64-bit
5. VM Name : vm1-oratony19 (puis répéter pour vm2-oratony19)
6. Processors : 8
7. Memory : 5120 MB
8. Network : Bridged
9. Disk : créer un disque principal pour /, /boot, /home, /tmp, /var
10. Ajouter deux disques supplémentaires :
    - 70 Go pour /opt/oracle/oradata
    - 30 Go pour /opt/oracle/orasave
```

### 1.4 Structure des répertoires Oracle (après installation)

```
/opt/oracle/
├── oradata/
│   └── ELIORA/
│       ├── adump/
│       ├── arch -> /opt/oracle/orasave/ELIORA/arch
│       ├── base/
│       ├── cdump/
│       ├── conf/
│       ├── create/
│       ├── diag/
│       ├── echange -> /opt/oracle/orasave/ELIORA/echange
│       ├── exp -> /opt/oracle/orasave/ELIORA/exp
│       ├── fra -> /opt/oracle/orasave/ELIORA/fra
│       ├── mirror -> /opt/oracle/orasave/ELIORA/mirror
│       ├── rman -> /opt/oracle/orasave/ELIORA/rman
│       ├── tmp/
│       ├── udump/
│       └── wallet/
└── orasave/
    └── ELIORA/
        ├── arch/       (archives redo logs)
        ├── echange/    (fichiers d'échange)
        ├── exp/        (exports)
        ├── fra/        (Fast Recovery Area)
        ├── mirror/     (miroir control files)
        └── rman/       (backups RMAN)
```

> **Bonne pratique** : Les répertoires `arch`, `fra`, `mirror`, `rman`, `echange`, `exp` sont des **liens symboliques** pointant vers le volume `/opt/oracle/orasave`. Cela sépare les fichiers de travail des fichiers de sauvegarde sur des volumes LVM distincts.

---

## 2. Installation d'Oracle Linux 8.10

### 2.1 Configuration lors de l'installation (Anaconda)

```
- Language : Français (ou Anglais selon préférence)
- Software Selection : Server (avec outils de développement recommandés)
- Partitionnement : Manuel (voir tableau section 1.2)
- Réseau : Activer l'interface réseau, définir le hostname
    - VM1 : vm1-oratony19
    - VM2 : vm2-oratony19
- Root password : définir un mot de passe fort
- Créer un utilisateur oracle (uid=54321, gid=oinstall)
```

### 2.2 Configuration post-installation

#### Fichier /etc/hosts (sur les deux VMs)

```bash
# /etc/hosts — à configurer identiquement sur vm1 et vm2
<IP_VM1>   vm1-oratony19
<IP_VM2>   vm2-oratony19
```

> Remplacer `<IP_VM1>` et `<IP_VM2>` par les IP réelles des VMs.

#### Paquets prérequis Oracle

```bash
# Installation des dépendances Oracle (en root)
dnf install -y oracle-database-preinstall-19c
```

Ce paquet configure automatiquement :
- Les paramètres kernel (`/etc/sysctl.conf`)
- Les limites système pour l'utilisateur oracle (`/etc/security/limits.conf`)
- Les groupes `oinstall`, `dba`, `oper`, `backupdba`, `dgdba`, `kmdba`

#### Variables d'environnement Oracle (dans ~/.bash_profile de l'utilisateur oracle)

```bash
# Oracle Environment
export ORACLE_BASE=/opt/oracle
export ORACLE_HOME=/opt/oracle/product/19.0.0/dbhome_1
export ORACLE_SID=ELIORA
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export NLS_DATE_FORMAT="DD/MM/YYYY HH24:MI:SS"
```

#### Désactivation du firewall et SELinux (environnement lab)

```bash
# Firewall
systemctl stop firewalld
systemctl disable firewalld

# SELinux — modifier /etc/selinux/config
sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
setenforce 0
```

---

## 3. Installation d'Oracle Database 19c

### 3.1 Prérequis avant installation

```bash
# Vérifier les groupes de l'utilisateur oracle
id oracle
# uid=54321(oracle) gid=54321(oinstall) groups=54321(oinstall),54322(dba),...

# Créer les répertoires Oracle
mkdir -p /opt/oracle/product/19.0.0/dbhome_1
mkdir -p /opt/oracle/oradata/ELIORA/{adump,base,cdump,conf,create,customize,diag,tmp,udump,wallet,logbook,adhoc}
mkdir -p /opt/oracle/orasave/ELIORA/{arch,echange,exp,fra,mirror,rman}

# Créer les liens symboliques
ln -s /opt/oracle/orasave/ELIORA/arch    /opt/oracle/oradata/ELIORA/arch
ln -s /opt/oracle/orasave/ELIORA/echange /opt/oracle/oradata/ELIORA/echange
ln -s /opt/oracle/orasave/ELIORA/exp     /opt/oracle/oradata/ELIORA/exp
ln -s /opt/oracle/orasave/ELIORA/fra     /opt/oracle/oradata/ELIORA/fra
ln -s /opt/oracle/orasave/ELIORA/mirror  /opt/oracle/oradata/ELIORA/mirror
ln -s /opt/oracle/orasave/ELIORA/rman    /opt/oracle/oradata/ELIORA/rman

# Changer le propriétaire
chown -R oracle:oinstall /opt/oracle
```

### 3.2 Décompression du logiciel Oracle 19c

```bash
# En tant qu'oracle, décompresser LINUX.X64_193000_db_home.zip dans ORACLE_HOME
cd /opt/oracle/product/19.0.0/dbhome_1
unzip /chemin/vers/LINUX.X64_193000_db_home.zip
```

### 3.3 Lancement de l'installation (mode silent)

```bash
# Installation silencieuse en tant qu'oracle
$ORACLE_HOME/runInstaller -silent \
  -responseFile $ORACLE_HOME/install/response/db_install.rsp \
  oracle.install.option=INSTALL_DB_SWONLY \
  ORACLE_HOSTNAME=vm1-oratony19 \
  UNIX_GROUP_NAME=oinstall \
  INVENTORY_LOCATION=/opt/oracle/oraInventory \
  ORACLE_HOME=/opt/oracle/product/19.0.0/dbhome_1 \
  ORACLE_BASE=/opt/oracle \
  oracle.install.db.InstallEdition=EE \
  oracle.install.db.OSDBA_GROUP=dba \
  oracle.install.db.OSBACKUPDBA_GROUP=backupdba \
  oracle.install.db.OSDGDBA_GROUP=dgdba \
  oracle.install.db.OSKMDBA_GROUP=kmdba \
  oracle.install.db.OSRACDBA_GROUP=racdba \
  SECURITY_UPDATES_VIA_MYORACLESUPPORT=false \
  DECLINE_SECURITY_UPDATES=true
```

#### Scripts root à exécuter après l'installation

```bash
# En tant que root
/opt/oracle/oraInventory/orainstRoot.sh
/opt/oracle/product/19.0.0/dbhome_1/root.sh
```

> **Note** : Ces scripts configurent les permissions et l'inventaire Oracle. À exécuter sur **chaque VM**.

---

## 4. Configuration Oracle Data Guard 19c Physical Standby

### 4.1 Architecture

```
┌─────────────────────┐         ┌─────────────────────┐
│   vm1-oratony19     │         │   vm2-oratony19     │
│   PRIMARY           │◄───────►│   PHYSICAL STANDBY  │
│   SID : ELIORA      │  Redo   │   SID : ELIORA      │
│   Service : ELIORA  │  Apply  │   Service : ELIORA  │
│   Port : 1521       │         │   _STBY Port : 1521 │
└─────────────────────┘         └─────────────────────┘
       MRP automatique activé sur la Standby
```

### 4.2 Création de la base Primary via DBCA

```bash
# Sur vm1-oratony19, en tant qu'oracle
dbca -silent \
  -createDatabase \
  -templateName General_Purpose.dbc \
  -gdbname ELIORA \
  -sid ELIORA \
  -responseFile NO_VALUE \
  -characterSet AL32UTF8 \
  -sysPassword <sys_password> \
  -systemPassword <system_password> \
  -createAsContainerDatabase false \
  -databaseType MULTIPURPOSE \
  -automaticMemoryManagement false \
  -totalMemory 2048 \
  -storageType FS \
  -datafileDestination /opt/oracle/oradata/ELIORA/base \
  -redoLogFileSize 200 \
  -emConfiguration NONE \
  -ignorePreReqs
```

### 4.3 Paramètres de la base Primary (spfile)

```sql
-- Se connecter en tant que SYSDBA sur vm1
sqlplus / as sysdba

-- Activer l'archivelog mode
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;

-- Paramètres Data Guard
ALTER SYSTEM SET LOG_ARCHIVE_CONFIG='DG_CONFIG=(ELIORA,ELIORA_STBY)' SCOPE=BOTH;
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1='LOCATION=/opt/oracle/oradata/ELIORA/arch VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=ELIORA' SCOPE=BOTH;
ALTER SYSTEM SET LOG_ARCHIVE_DEST_2='SERVICE=ELIORA_STBY LGWR ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=ELIORA_STBY' SCOPE=BOTH;
ALTER SYSTEM SET LOG_ARCHIVE_DEST_STATE_1=ENABLE SCOPE=BOTH;
ALTER SYSTEM SET LOG_ARCHIVE_DEST_STATE_2=ENABLE SCOPE=BOTH;
ALTER SYSTEM SET LOG_ARCHIVE_FORMAT='%t_%s_%r.arc' SCOPE=SPFILE;
ALTER SYSTEM SET LOG_ARCHIVE_MAX_PROCESSES=4 SCOPE=BOTH;
ALTER SYSTEM SET DB_UNIQUE_NAME='ELIORA' SCOPE=SPFILE;
ALTER SYSTEM SET FAL_SERVER='ELIORA_STBY' SCOPE=BOTH;
ALTER SYSTEM SET FAL_CLIENT='ELIORA' SCOPE=BOTH;
ALTER SYSTEM SET STANDBY_FILE_MANAGEMENT=AUTO SCOPE=BOTH;
ALTER SYSTEM SET DB_FILE_NAME_CONVERT='/opt/oracle/oradata/ELIORA','/opt/oracle/oradata/ELIORA' SCOPE=SPFILE;
ALTER SYSTEM SET LOG_FILE_NAME_CONVERT='/opt/oracle/oradata/ELIORA','/opt/oracle/oradata/ELIORA' SCOPE=SPFILE;

-- Vérifier que la base est en archivelog
ARCHIVE LOG LIST;
```

### 4.4 Configuration du listener et TNS (sur les deux VMs)

#### $ORACLE_HOME/network/admin/listener.ora

```ini
# Sur vm1-oratony19
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = vm1-oratony19)(PORT = 1521))
    )
  )

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = ELIORA)
      (ORACLE_HOME = /opt/oracle/product/19.0.0/dbhome_1)
      (SID_NAME = ELIORA)
    )
  )
```

```ini
# Sur vm2-oratony19
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = vm2-oratony19)(PORT = 1521))
    )
  )

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = ELIORA_STBY)
      (ORACLE_HOME = /opt/oracle/product/19.0.0/dbhome_1)
      (SID_NAME = ELIORA)
    )
  )
```

#### $ORACLE_HOME/network/admin/tnsnames.ora (identique sur les deux VMs)

```ini
ELIORA =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = vm1-oratony19)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ELIORA)
    )
  )

ELIORA_STBY =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = vm2-oratony19)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ELIORA_STBY)
    )
  )
```

#### Démarrage et vérification du listener

```bash
# Sur chaque VM
lsnrctl start
lsnrctl status

# Test de connectivité TNS
tnsping ELIORA
tnsping ELIORA_STBY
```

### 4.5 Création de la Standby via RMAN DUPLICATE

```bash
# Sur vm2-oratony19 — démarrer la base en NOMOUNT
sqlplus / as sysdba
STARTUP NOMOUNT;
EXIT;

# Depuis vm1 ou vm2 — dupliquer la base pour créer la standby
rman TARGET sys/<password>@ELIORA AUXILIARY sys/<password>@ELIORA_STBY

# Dans RMAN
DUPLICATE TARGET DATABASE
  FOR STANDBY
  FROM ACTIVE DATABASE
  DORECOVER
  SPFILE
    SET db_unique_name='ELIORA_STBY'
    SET LOG_ARCHIVE_DEST_1='LOCATION=/opt/oracle/oradata/ELIORA/arch VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=ELIORA_STBY'
    SET LOG_ARCHIVE_DEST_2='SERVICE=ELIORA LGWR ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=ELIORA'
    SET FAL_SERVER='ELIORA'
    SET FAL_CLIENT='ELIORA_STBY'
  NOFILENAMECHECK;
```

### 4.6 Activation du MRP (Managed Recovery Process) en mode automatique

```sql
-- Sur vm2-oratony19 (Standby), en tant que SYSDBA
sqlplus / as sysdba

-- Activer la récupération gérée en temps réel (Real-Time Apply)
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE
  USING CURRENT LOGFILE DISCONNECT FROM SESSION;

-- Vérifier le statut du MRP
SELECT PROCESS, STATUS, THREAD#, SEQUENCE#, BLOCK#, BLOCKS
FROM V$MANAGED_STANDBY
WHERE PROCESS = 'MRP0';
```

### 4.7 Vérifications post-configuration

#### Sur la Primary (vm1-oratony19)

```sql
sqlplus / as sysdba

-- Vérifier la configuration Data Guard
SELECT NAME, DB_UNIQUE_NAME, DATABASE_ROLE, OPEN_MODE, PROTECTION_MODE
FROM V$DATABASE;

-- Vérifier l'envoi des archives
SELECT DEST_ID, STATUS, TARGET, ARCHIVER, SCHEDULE, DESTINATION
FROM V$ARCHIVE_DEST
WHERE STATUS = 'VALID';

-- Forcer un switch de log pour tester la transmission
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM ARCHIVE LOG CURRENT;

-- Vérifier la progression de l'apply sur la standby
SELECT SEQUENCE#, APPLIED FROM V$ARCHIVED_LOG ORDER BY SEQUENCE# DESC;
```

#### Sur la Standby (vm2-oratony19)

```sql
sqlplus / as sysdba

-- Vérifier le rôle de la base
SELECT NAME, DB_UNIQUE_NAME, DATABASE_ROLE, OPEN_MODE
FROM V$DATABASE;
-- Résultat attendu : DATABASE_ROLE = 'PHYSICAL STANDBY', OPEN_MODE = 'MOUNTED'

-- Vérifier que le MRP applique bien les archives
SELECT PROCESS, STATUS, SEQUENCE#
FROM V$MANAGED_STANDBY
WHERE PROCESS IN ('MRP0', 'RFS');

-- Vérifier le gap (retard éventuel)
SELECT * FROM V$ARCHIVE_GAP;
-- Résultat attendu : aucune ligne (pas de gap)
```

#### Vérification globale depuis la Primary

```sql
-- Vue Data Guard Broker (si configuré) ou vérification manuelle
SELECT NAME, VALUE, UNIT, TIME_COMPUTED
FROM V$DATAGUARD_STATS
WHERE NAME IN ('transport lag', 'apply lag');
-- Résultat attendu : lag = 0 ou très proche de 0
```

### 4.8 Commandes utiles au quotidien

```sql
-- === PRIMARY ===

-- Forcer un log switch
ALTER SYSTEM SWITCH LOGFILE;

-- Vérifier les destinations d'archivage
SELECT DEST_ID, STATUS, DESTINATION FROM V$ARCHIVE_DEST WHERE STATUS='VALID';

-- === STANDBY ===

-- Arrêter le MRP
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;

-- Redémarrer le MRP
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE
  USING CURRENT LOGFILE DISCONNECT FROM SESSION;

-- Ouvrir la standby en lecture seule (pour reporting)
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
ALTER DATABASE OPEN READ ONLY;

-- Remettre en récupération managed
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE
  USING CURRENT LOGFILE DISCONNECT FROM SESSION;
```

---

## Résumé des points clés

| Point | Valeur / Décision |
|---|---|
| Mode de protection | Maximum Performance (défaut) |
| MRP | Automatique avec Real-Time Apply |
| Archivelog | Activé sur la Primary |
| STANDBY_FILE_MANAGEMENT | AUTO |
| FRA | /opt/oracle/orasave/ELIORA/fra |
| Archives | /opt/oracle/orasave/ELIORA/arch |
| Réseau VM | Bridged (communication directe entre VMs) |
| Nom unique Primary | ELIORA |
| Nom unique Standby | ELIORA_STBY |

---

*Guide rédigé à partir du lab personnel de Tony — Juin 2026*
