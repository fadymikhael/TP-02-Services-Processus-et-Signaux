
# TP 02 : Services, Processus et Signaux

## 1. Secure Shell (SSH)

### 1.1 Exercice : Connexion SSH en tant que root

J'ai modifié le fichier de configuration SSH pour permettre les connexions distantes en tant que root en activant deux options :

1. **PermitRootLogin yes** : 
   - **Avantage** : Permet de se connecter directement en tant que root, facilitant ainsi l'administration.
   - **Inconvénient** : Dangereux en production, car il expose le serveur aux attaques par force brute sur le compte root.

2. **PasswordAuthentication yes** : 
   - **Avantage** : Plus simple à configurer, ne nécessitant pas de clés SSH au début.
   - **Inconvénient** : Moins sécurisé que l'authentification par clé SSH, les mots de passe pouvant être devinés ou volés.

### Quand utiliser ces options ?
- **PermitRootLogin yes** : Utilisé dans un environnement de développement ou pour une configuration rapide.
- **PermitRootLogin no** : Recommandé en production pour limiter l'accès root et renforcer la sécurité.
- **PasswordAuthentication yes** : Utilisé temporairement avant la configuration des clés SSH.
- **PasswordAuthentication no** : Idéal en production pour forcer l'utilisation de clés SSH.

### Générer une paire de clés SSH
J'ai utilisé la commande suivante pour générer une paire de clés SSH :

```bash
ssh-keygen -t rsa -b 4096
```

- **-t rsa** : Utilise l'algorithme RSA.
- **-b 4096** : Spécifie une taille de clé de 4096 bits.

**Résultat** :
```bash
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
```

### Copier la clé publique sur le serveur
Pour copier la clé publique vers le serveur, utilisez la commande :

```bash
ssh-copy-id root@10.0.2.15
```

### Résolution des problèmes de connexion ( mon cas )

Lorsque j'ai redémarré la machine virtuelle, j'étais en mode NAT et la connexion a échoué. Voici ce que j'ai fait pour résoudre ce problème :

```bash
dhclient -r enp0s3    # Retirer l'adresse IP
dhclient enp0s3       # Demander une nouvelle adresse IP
```

**Résultat** : J'ai pu me reconnecter à la machine virtuelle depuis ma machine hôte avec la nouvelle adresse IP.

---

## 2. Processus et Signaux

### 2.1 Affichage des processus
Pour afficher les processus en cours avec leur utilisation CPU et mémoire :

```bash
ps -eo user,pid,%cpu,%mem,stat,start,time,command
```

**Résultat** :
```bash
USER         PID %CPU %MEM STAT  STARTED     TIME COMMAND
root           1  0.4  0.3 Ss   19:19:12 00:00:01 /sbin/init
root         556 66.6  0.1 R+   19:22:59 00:00:00 ps -eo user,pid,%cpu,%mem,stat,start,time,command
```

- **TIME** : Temps CPU total utilisé par le processus.
- **Le processus utilisant le plus de CPU** : Le processus avec PID 556, la commande `ps`, a temporairement utilisé 66.6% du CPU.
- **Premier processus lancé** : Le processus avec PID 1, `/sbin/init`, est le premier processus lancé lors du démarrage du système.


### Affichage des ancêtres d'un processus
Pour afficher la hiérarchie des processus :

```bash
ps -eo pid,ppid,command | grep [p]s
```

**Résultat** :
```bash
    504       1 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
    568     553 ps -eo pid,ppid,command
```

### Utilisation de `pstree`
Pour afficher l'arborescence des processus :

```bash
pstree -p
```

**Résultat** :
```bash
systemd(1)─┬─cron(442)
           ├─sshd(504)───sshd(547)───bash(553)───pstree(801)
```

---

## 3. Gestion des tâches en arrière-plan

### Création des scripts `date.sh` et `date-toto.sh`

1. **Script `date.sh`** :

```bash
#!/bin/sh
while true; do
  sleep 1
  echo -n 'date '
  date +%T
done
```

2. **Script `date-toto.sh`** :

```bash
#!/bin/sh
while true; do
  sleep 1
  echo -n 'toto '
  date --date '5 hour ago' +%T
done
```

### Exécution des scripts

Lancer les scripts en arrière-plan et les suspendre :

```bash
./date.sh      # Lancer le script
CTRL+Z         # Suspendre
./date-toto.sh # Lancer l'autre script
```

**Résultat** :
```bash
date 01:39:41
date 01:39:42
date 01:39:43
^Z
[1]+  Stopped ./date.sh

toto 20:40:06
toto 20:40:07
toto 20:40:08
^Z
[2]+  Stopped ./date-toto.sh
```

Reprendre un script :

```bash
fg %1  # Reprendre date.sh
fg %2  # Reprendre date-toto.sh
```

### Arrêter les scripts avec `kill`

Lister les processus et tuer les scripts :

```bash
ps aux | grep date.sh
```

**Résultat** :
```bash
root         633  0.0  0.0   2576   916 pts/0    T    01:39   0:00 /bin/sh ./date.sh
root         691  0.0  0.0   6332  2052 pts/0    S+   01:43   0:00 grep date.sh
```

Pour arrêter le script :
```bash
kill -9 633
```

**Résultat** :
```bash
[1]+  Killed ./date.sh
```

---

## 4. Les tubes : `cat` et `tee`

### Différence entre `cat` et `tee`
- **cat** : Affiche le contenu d'un fichier ou d'une commande.
- **tee** : Affiche et enregistre la sortie d'une commande dans un fichier.

### Exemples de commandes

1. **`ls | cat`** :
   - Résultat : La liste des fichiers est affichée dans le terminal.
   
2. **`ls -l | cat > liste`** :
   - Résultat : Le fichier `liste` contient la sortie de `ls -l`. Rien n'est affiché dans le terminal.

3. **`ls -l | tee liste`** :
   - Résultat : La sortie est affichée dans le terminal et enregistrée dans le fichier `liste`.

4. **`ls -l | tee liste | wc -l`** :
   - Résultat : Le nombre de lignes de la sortie est affiché et le fichier `liste` est mis à jour.

---

## 5. Journal système : `rsyslog`

### Vérifier si `rsyslog` est lancé

```bash
ps aux | grep rsyslog
```

**Résultat** :
```bash
root         980  0.0  0.1 221788  6344 ?        Ssl  02:11   0:00 /usr/sbin/rsyslogd -n -iNONE
```

Le service `rsyslog` est bien lancé, avec le PID 980.

### Fichiers de log importants

- **/var/log/syslog** : Contient les messages des services standards.
- **/var/log/kern.log** : Contient les messages du noyau.

---

## 6. Détails du matériel détecté

### Informations sur le processeur

Le modèle du processeur détecté est un **Intel Core i7-10510U**. Pour obtenir plus de détails, utilisez la commande :

```bash
dmesg | grep -i "cpu"
```

**Résultat** :
```bash
[    1.354843] smpboot: CPU0: Intel(R) Core(TM) i7-10510U CPU @ 1.80GHz
```

### Informations sur la carte réseau

Pour obtenir des informations sur la carte réseau :

```bash
dmesg | grep -i "eth"
```

**Résultat** :
```bash
[    5.148563] e1000 0000:00:03.0 eth0: Intel(R) PRO/1000 Network Connection
[    5.182687]

