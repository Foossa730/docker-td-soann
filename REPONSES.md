# TD1 — Prise en main — Réponses

Environnement : macOS (Apple Silicon, arm64), Docker Desktop 4.94 (Engine 29.8.2), terminal zsh.

---

## Partie A — Premiers conteneurs

### A1

```bash
docker run hello-world
docker run hello-world
```

**Observation :** à la première exécution, Docker affiche `Unable to find image 'hello-world:latest' locally`,
puis `Pulling from library/hello-world … Pull complete … Downloaded newer image`, et enfin le message « Hello from Docker! ».
À la seconde exécution, le message s'affiche directement, sans aucun téléchargement.

**Explication :** la première fois, l'image n'existait pas dans le cache local : le daemon l'a téléchargée depuis
Docker Hub (variante `arm64v8` pour mon Mac). La seconde fois, l'image est déjà présente localement, Docker crée
simplement un **nouveau** conteneur à partir d'elle. Il y a donc eu deux conteneurs différents (visibles avec
`docker ps -a`), mais une seule image.

### A2

```bash
docker run -d --name web1 -p 8080:80 nginx:1.29-alpine
docker run -d --name web2 -p 8081:80 nginx:1.29-alpine
docker ps
```

`docker ps` montre `0.0.0.0:8080->80/tcp` pour web1 et `0.0.0.0:8081->80/tcp` pour web2 ; les deux pages
http://localhost:8080 et http://localhost:8081 affichent « Welcome to nginx! ».

**Pourquoi pas de conflit ?** Chaque conteneur a son propre espace réseau isolé (namespace réseau) avec sa propre
interface et sa propre IP : le port 80 de web1 et le port 80 de web2 ne sont pas le même port. Le seul endroit où
un conflit est possible est la **machine hôte** : `-p 8080:80` associe le port 8080 de l'hôte au port 80 du conteneur.
Comme 8080 et 8081 sont deux ports différents sur l'hôte, tout va bien.

**Et si web2 est aussi publié sur 8080 ?**

```bash
docker run -d --name web3 -p 8080:80 nginx:1.29-alpine
```

→ `Bind for 0.0.0.0:8080 failed: port is already allocated`. Le port 8080 de l'hôte est déjà pris par web1.
Le conteneur est quand même **créé** (statut `Created` dans `docker ps -a`) mais ne démarre pas ; je l'ai supprimé
avec `docker rm web3`.

### A3

```bash
docker logs web1          # afficher les logs
docker logs -f web1       # les suivre en continu (Ctrl+C pour arrêter)
```

**Observation :** une ligne par requête, au format « access log » de nginx :

```
192.168.65.1 - - [06/Oct/2026:07:58:19 +0000] "GET / HTTP/1.1" 200 896 "-" "curl/8.7.1" "-"
```

En mode `-f`, les nouvelles lignes apparaissent en direct à chaque rafraîchissement. Une URL inexistante
(`/suivi`) produit en plus une ligne `[error] … open() "/usr/share/nginx/html/suivi" failed`.

**D'où viennent ces lignes ?** Docker capture la **sortie standard (stdout) et la sortie d'erreur (stderr)** du
processus principal du conteneur. Dans l'image nginx officielle, les fichiers de log sont des liens vers ces sorties :

```bash
docker exec web1 ls -l /var/log/nginx/
# access.log -> /dev/stdout
# error.log  -> /dev/stderr
```

Les accès vont donc sur stdout et les erreurs sur stderr, et `docker logs` les relit. (L'IP `192.168.65.1` est
celle de la VM de Docker Desktop, qui relaie la requête venant de mon navigateur.)

### A4

```bash
docker exec -it web1 sh
# dans le conteneur :
echo Soann > /usr/share/nginx/html/index.html
exit
```

Le navigateur (et `curl localhost:8080`) affiche bien `Soann`.

```bash
docker rm -f web1
docker run -d --name web1 -p 8080:80 nginx:1.29-alpine
```

→ La page affiche de nouveau « Welcome to nginx! ».

**Où est passée la modification ?** Elle avait été écrite dans la **couche inscriptible du conteneur**, qui est
détruite en même temps que lui par `docker rm`. Le nouveau conteneur repart de l'image `nginx:1.29-alpine`,
qui, elle, n'a jamais été modifiée (elle est en lecture seule).

**Conclusion :** `docker exec` est utile pour **diagnostiquer** (regarder un fichier, tester une commande),
mais pas pour modifier une application : les changements sont éphémères, non tracés et non reproductibles.
Pour un changement durable, il faut le mettre dans l'image (Dockerfile) ou dans un volume.

---

## Partie B — Variables d'environnement et mode interactif

### B1

```bash
docker run --rm -e PRENOM=Soann alpine printenv PRENOM   # → Soann
docker run --rm alpine printenv PRENOM                    # → (rien), code de sortie 1
echo $PRENOM                                              # → (ligne vide)
```

**Observation :** avec `-e`, le conteneur affiche `Soann`. Sans `-e`, rien ne s'affiche et `printenv` sort en
code 1 (variable inexistante). Sur ma machine, `echo $PRENOM` est vide : la variable n'existe pas sur l'hôte.

**Où vit la variable ?** Uniquement dans l'**environnement du processus du conteneur** : Docker l'injecte au
démarrage du conteneur. Elle n'est ni dans l'image, ni sur l'hôte, et disparaît avec le conteneur (ici, `--rm`).

### B2

```bash
docker run --rm -it alpine sh
# dans le conteneur :
apk add curl
curl --version      # → curl 8.22.0 (aarch64-alpine-linux-musl) …  : ça marche
exit
docker run --rm -it alpine sh
which curl          # → rien : curl est absent
```

**`curl` est-il encore là ?** Non. L'installation a été faite dans le **conteneur** (sa couche inscriptible),
pas dans l'image `alpine`. Avec `--rm`, ce conteneur a été supprimé en quittant ; la nouvelle commande crée un
conteneur neuf à partir de l'image d'origine, qui ne contient pas curl.
(Pour l'installer durablement : `RUN apk add curl` dans un Dockerfile, pour construire une nouvelle image.)

---

## Partie C — Images et couches

### C1

```bash
docker pull node:24 ; docker pull node:24-slim ; docker pull node:24-alpine
docker image ls node
docker run --rm node:24 sh -c 'ls /usr/bin | wc -l'
docker run --rm node:24 which gcc git curl
# (idem avec node:24-slim et node:24-alpine)
```

| Image            | Taille  | Commandes dans `/usr/bin` | gcc | git | curl | Version Node |
|------------------|---------|---------------------------|-----|-----|------|--------------|
| `node:24`        | 1,65 GB | 664                       | oui | oui | oui  | v24.21.0     |
| `node:24-slim`   | 351 MB  | 273                       | non | non | non  | v24.21.0     |
| `node:24-alpine` | 238 MB  | 143                       | non | non | non  | v24.21.0     |

**Que contient la plus grosse en plus ?** `node:24` est basée sur Debian complet (image « buildpack-deps ») :
elle embarque toute une chaîne de compilation (gcc, make…), des outils de développement (git, curl, ssh…)
et de nombreuses bibliothèques de développement. `-slim` est une Debian minimale, `-alpine` une distribution
minuscule basée sur musl et BusyBox.

**Ces outils sont-ils utiles pour faire tourner une API ?** Non. Pour **exécuter** une API Node, il suffit du
runtime `node` et des dépendances `node_modules`. Les compilateurs ou git peuvent servir au **build**
(ex. modules natifs), mais en production ils alourdissent l'image (téléchargement, stockage) et augmentent la
surface d'attaque. On privilégie donc `slim` ou `alpine` pour l'exécution (éventuellement avec un build multi-étapes).

### C2

```bash
docker history node:24-alpine
```

```
SIZE     CREATED BY
0B       CMD ["node"]
0B       ENTRYPOINT ["docker-entrypoint.sh"]
20.5kB   COPY docker-entrypoint.sh /usr/local/bin/
5.48MB   RUN … apk add … yarn …
0B       ENV YARN_VERSION=1.22.22
161MB    RUN addgroup -g 1000 node && adduser … && apk add libstdc++ … (installation de Node)
0B       ENV NODE_VERSION=24.21.0
0B       CMD ["/bin/sh"]
9.31MB   ADD alpine-minirootfs-3.24.2-aarch64.tar.gz /
```

**Combien de couches ?** L'historique liste **9 étapes**, mais seules celles qui modifient le système de fichiers
produisent une vraie couche : **4 couches** (`ADD` de la base Alpine, `RUN` Node, `RUN` Yarn, `COPY` de
l'entrypoint), ce que confirme `docker image inspect -f '{{len .RootFS.Layers}}' node:24-alpine` → `4`.
Les `ENV`, `CMD`, `ENTRYPOINT` (0 B) ne font qu'ajouter des métadonnées.

**La plus lourde :** l'instruction `RUN` qui crée l'utilisateur `node` et **installe Node.js** (161 MB).

### C3

```bash
docker image inspect nginx:1.29-alpine | grep -A4 -E '"Cmd"|"ExposedPorts"'
```

```json
"Cmd": ["nginx", "-g", "daemon off;"],
"Entrypoint": ["/docker-entrypoint.sh"],
"ExposedPorts": { "80/tcp": {} },
```

**Commande au démarrage :** `nginx -g "daemon off;"` (passée en argument à l'entrypoint `/docker-entrypoint.sh`).
Le `daemon off;` garde nginx au premier plan : il reste le processus principal, sinon le conteneur s'arrêterait.

**Port indiqué :** `80/tcp`. C'est cohérent avec A2, où j'ai publié `-p 8080:80` : le port **de droite** (celui du
conteneur) est bien 80. `ExposedPorts` est de la documentation : il n'ouvre rien tout seul, c'est `-p` qui publie.

---

## Partie D — Énigmes

### D1 — `docker run -d alpine` ne laisse rien dans `docker ps`

**Observation :**

```bash
docker run -d alpine          # renvoie un ID
docker ps                     # vide
docker ps -a                  # alpine  "/bin/sh"  Exited (0) 1 second ago
docker image inspect -f '{{json .Config.Cmd}}' alpine   # ["/bin/sh"]
```

**Explication :** un conteneur vit tant que son processus principal (PID 1) vit. Le `Cmd` d'alpine est `/bin/sh`.
Lancé avec `-d` sans `-it`, le shell n'a aucune entrée (stdin fermé) : il lit « fin de fichier », n'a rien à faire
et se termine immédiatement avec le code 0. Le conteneur passe donc à `Exited (0)` : `docker ps` ne montre que
les conteneurs en cours, il faut `-a` pour le voir.

**Correction :** donner au conteneur un processus qui dure, par exemple :

```bash
docker run -d alpine sleep 300      # → "Up" dans docker ps
# ou, pour un shell qu'on peut rejoindre : docker run -dit alpine
```

### D2 — `-p 9082:8080` : le conteneur tourne mais la page ne répond pas

**Observation :**

```bash
docker run -d -p 9082:8080 nginx:1.29-alpine
curl localhost:9082        # curl: (56) Recv failure: Connection reset by peer
docker port <id>           # 8080/tcp -> 0.0.0.0:9082
docker exec <id> netstat -tln   # nginx écoute sur 0.0.0.0:80 seulement
```

**Explication :** dans `-p HÔTE:CONTENEUR`, le port de droite est celui **du conteneur**. Docker redirige bien
9082 de l'hôte vers le port 8080 du conteneur… mais nginx n'écoute pas sur 8080 : il écoute sur **80**
(cf. `ExposedPorts` en C3). La connexion arrive sur un port où personne n'écoute et est refusée.

**Correction :**

```bash
docker run -d -p 9082:80 nginx:1.29-alpine   # → "Welcome to nginx!" sur http://localhost:9082
```

### D3 — Arrêt lent de `sleep`

```bash
docker run -d --name dormeur alpine sleep 1000
time docker stop dormeur                     # → 3,126 s au total
docker ps -a --filter name=dormeur           # → Exited (137)
```

**Observation :** l'arrêt a pris environ **3,1 s** sur ma machine (le délai de grâce dépend de la configuration ;
il est de 10 s par défaut), et le code de sortie est **137**.

**Explication (cycle de vie) :** `docker stop` envoie d'abord **SIGTERM** au PID 1 du conteneur pour lui demander de
s'arrêter proprement, puis attend la fin du délai de grâce. Ici `sleep` est le PID 1 et n'a pas réagi au SIGTERM.
À l'expiration du délai, Docker envoie **SIGKILL**, qui ne peut pas être ignoré : le processus est tué brutalement.
137 = 128 + 9, 9 étant le numéro de SIGKILL. Le temps mesuré correspond donc à l'attente inutile du délai de grâce.

**Bonus — avec `--init` :**

```bash
docker run -d --init --name dormeur2 alpine sleep 1000
docker exec dormeur2 ps         # PID 1 = /sbin/docker-init -- sleep 1000 ; sleep a le PID 7
time docker stop dormeur2       # → 0,075 s
docker ps -a --filter name=dormeur2   # → Exited (143)
```

Avec `--init`, un mini-init (`docker-init`, basé sur tini) devient le PID 1 et `sleep` devient un processus enfant.
L'init relaie le SIGTERM à `sleep`, qui meurt immédiatement : l'arrêt est **quasi instantané** et le code de sortie est
**143** = 128 + 15 (SIGTERM), soit un arrêt propre au lieu d'un kill forcé.

### D4 — Le conteneur « gourmand »

```bash
docker run --name gourmand --memory 50m node:24-alpine \
  node -e "const a=[]; while(true) a.push(new Array(1e6).fill(1))"
echo $?                         # → 137
docker inspect gourmand | grep -E '"OOMKilled"|"ExitCode"|"Memory"'
#   "OOMKilled": true,
#   "ExitCode": 137,
#   "Memory": 52428800,         (= 50 Mio)
```

**Que se passe-t-il ?** Le programme alloue de la mémoire en boucle. En moins d'une seconde, il atteint la limite de
50 Mo fixée par `--memory 50m` et le conteneur s'arrête brutalement, sans message d'erreur de Node.

**Code de sortie :** 137 (128 + 9 → tué par SIGKILL).

**Champ qui le confirme :** `State.OOMKilled: true` dans `docker inspect gourmand`.

**Mécanisme du noyau :** les **cgroups** (control groups), qui limitent les ressources (ici la mémoire) d'un groupe de
processus. Quand le cgroup du conteneur dépasse sa limite, l'**OOM killer** (Out Of Memory) du noyau Linux tue le
processus avec SIGKILL.

---

## Partie E — Ménage

```bash
docker system df                 # espace avant
docker ps -a                     # liste des conteneurs du TD
docker container prune -f        # supprime tous les conteneurs arrêtés
docker system df                 # espace après
```

| `docker system df` | Avant                         | Après                 |
|--------------------|-------------------------------|-----------------------|
| Images             | 6 — 2,329 GB                  | 6 — 2,329 GB          |
| Conteneurs         | 11 (5 actifs) — 360,4 kB      | 5 (5 actifs) — 331,8 kB |
| Volumes            | 0                             | 0                     |

`docker container prune -f` a supprimé les 6 conteneurs arrêtés (`gourmand`, `dormeur`, `dormeur2`, `d1` et les
deux `hello-world`) : **28,67 kB récupérés**. C'est très peu, car un conteneur ne stocke que sa fine couche
inscriptible ; l'essentiel de l'espace (2,3 GB) est occupé par les **images**, surtout `node:24` (1,65 GB).
J'ai ensuite supprimé les conteneurs encore actifs du TD avec `docker rm -f web1 web2 …` (→ 0 conteneur).
Pour récupérer l'espace des images, on pourrait faire `docker image prune -a` (ou `docker rmi node:24`).
