# Webserv — Fiche conceptuelle exhaustive

*Comprendre le projet de bout en bout : le **pourquoi** de chaque mécanisme, pas seulement le **quoi**. Alignée sur le sujet officiel **Webserv v24.0**.*

> **La phrase qui résume tout :** écrire, en **C++98**, un serveur HTTP **mono-thread**, **non-bloquant**, piloté par **un seul `poll()`**, capable de servir un **vrai navigateur** façon NGINX, et qui **ne crash jamais**.

---

## Sommaire

1. [HTTP — la matière](#1-http--la-matière)
2. [Le modèle d'exécution — le cœur](#2-le-modèle-dexécution--le-cœur)
3. [Sockets & réseau](#3-sockets--réseau)
4. [La vie d'une requête (bout en bout)](#4-la-vie-dune-requête-bout-en-bout)
5. [Parsing HTTP en détail](#5-parsing-http-en-détail)
6. [Le fichier de configuration](#6-le-fichier-de-configuration)
7. [Routing & méthodes](#7-routing--méthodes)
8. [CGI — le contenu dynamique](#8-cgi--le-contenu-dynamique)
9. [Robustesse](#9-robustesse)
10. [Contraintes & règles de compliance](#10-contraintes--règles-de-compliance)
11. [Livrables annexes](#11-livrables-annexes)
12. [Répartition binôme & structures partagées](#12-répartition-binôme--structures-partagées)
13. [Ordre d'attaque](#13-ordre-dattaque)
14. [Checklist des pièges à 0](#14-checklist-des-pièges-à-0)

---

## 1. HTTP — la matière

### C'est quoi, au fond
HTTP est un protocole **texte**, **requête/réponse**, **sans état** (stateless). Un client (navigateur) ouvre une connexion TCP, envoie une **requête**, le serveur renvoie une **réponse**, et c'est tout. Ton serveur ne « comprend » pas le web — il lit des octets, les interprète selon des règles précises, et renvoie des octets valides. Tout le projet, c'est **transformer un flux d'octets en requête structurée, puis produire une réponse structurée**.

### Anatomie d'une requête
```
GET /index.html HTTP/1.1\r\n      ← ligne de requête : MÉTHODE  URI  VERSION
Host: localhost:8080\r\n          ← en-têtes (clé: valeur), un par ligne
Content-Length: 0\r\n
\r\n                              ← LIGNE VIDE = fin des en-têtes
[corps optionnel]                 ← présent surtout pour POST
```
Points clés :
- Les fins de ligne sont **`\r\n`** (CRLF), pas `\n`. La séparation en-têtes/corps, c'est **`\r\n\r\n`**.
- L'**URI** peut contenir un chemin + une query string : `/search?q=chat` → path `/search`, query `q=chat`.
- `Host` est **obligatoire** en HTTP/1.1 (le navigateur l'envoie toujours).

### Anatomie d'une réponse
```
HTTP/1.1 200 OK\r\n               ← ligne de statut : VERSION  CODE  RAISON
Content-Type: text/html\r\n       ← en-têtes
Content-Length: 137\r\n
\r\n
<html>…</html>                    ← corps
```
- **`Content-Length`** dit au client combien d'octets de corps lire. Si tu te trompes, le navigateur bloque ou tronque. C'est **critique**.
- `Content-Type` (MIME) dit au navigateur comment interpréter le corps (`text/html`, `image/png`, `text/css`…).

### Les en-têtes qui comptent vraiment
| En-tête | Sens | Pourquoi tu t'en soucies |
|---|---|---|
| `Host` | quel virtual host | présent partout ; obligatoire HTTP/1.1 |
| `Content-Length` | taille du corps en octets | sais quand le corps est **complet** (requête) / bien formé (réponse) |
| `Transfer-Encoding: chunked` | corps envoyé en morceaux, sans taille connue d'avance | tu dois **dé-chunker** |
| `Connection: keep-alive` / `close` | garder ou fermer la connexion après | HTTP/1.1 = keep-alive par défaut |
| `Content-Type` | type MIME | à mettre correctement dans tes réponses |
| `Location` | URL de redirection | pour les 3xx |

### Méthodes obligatoires
**GET, POST, DELETE** (au minimum). Une méthode non autorisée sur une route → **405**. Une méthode que tu ne gères pas → **501** (ou 405 selon config).

### Codes de statut à maîtriser
`200 OK`, `201 Created`, `204 No Content`, `301/302` (redirections), `400 Bad Request` (requête malformée), `403 Forbidden`, `404 Not Found`, `405 Method Not Allowed`, `413 Payload Too Large` (corps > `client_max_body_size`), `500 Internal Server Error`, `501 Not Implemented`. **Le sujet exige des codes de statut *exacts*.**

### ⚠️ La version HTTP
Le sujet dit littéralement : *« HTTP 1.0 suggéré comme référence, mais non imposé »*. En pratique tu **vises HTTP/1.1**, parce que les navigateurs l'utilisent (`Host`, keep-alive, chunked). Lis les RFC (7230–7235 ou l'ancienne 2616), et **teste avec `telnet` et `NGINX`** avant de coder — le sujet insiste là-dessus.

---

## 2. Le modèle d'exécution — le cœur

C'est **le** morceau du projet, le plus dur et le plus scruté. Tout repose sur une seule idée.

### Tout est un descripteur de fichier (fd)
Sous Unix, une socket d'écoute, une connexion client, un pipe vers un CGI : tout est un **fd**, un simple entier. Le serveur ne fait que **surveiller un ensemble de fd** et réagir quand l'un devient prêt.

### La boucle événementielle
Un seul thread, une boucle infinie autour d'**un seul `poll()`**. `poll()` prend la liste de *tous* tes fd et **bloque jusqu'à ce qu'au moins un soit prêt** (lisible ou inscriptible), puis te rend la liste des prêts.

```
setup :
    parse la conf
    pour chaque paire host:port à écouter :
        socket() → setsockopt(SO_REUSEADDR) → bind() → listen()
        passe le fd en non-bloquant
        ajoute-le au poll (surveillé en LECTURE)

boucle infinie :
    poll(tous les fd)                  // bloque jusqu'à ≥1 fd prêt
    pour chaque fd prêt :

        si c'est un socket d'écoute (prêt en lecture) :
            accept() → nouveau fd client → non-bloquant → ajoute au poll (LECTURE)

        si c'est un client prêt en LECTURE :
            recv() UN bloc → ajoute au buffer d'entrée du client
            essaie de parser : requête complète ? → traite, prépare la réponse,
                               passe le client en surveillance ÉCRITURE

        si c'est un client prêt en ÉCRITURE :
            send() UN bloc du buffer de sortie
            tout envoyé ? → keep-alive (reset) ou close()
```

### Le « pourquoi » du non-bloquant
Avec **un seul thread**, si tu fais un `recv()` **bloquant** sur un client lent (ou malveillant qui envoie 1 octet par minute), **tout le serveur gèle** en attendant. Inacceptable. Donc :
- **Tous les fd réseau sont non-bloquants** (`fcntl(fd, F_SETFL, O_NONBLOCK)`).
- `poll()` sert d'**aiguilleur** : il ne te réveille que sur les fd réellement prêts.
- Tu sers **un petit bout** de chaque fd prêt, puis tu **reboucles**. Ainsi N clients progressent « en même temps » sur un seul thread, sans qu'aucun n'en bloque un autre.

### Les règles d'or (⚠️ chacune = fail direct si violée)
1. **Un seul `poll()`** (ou `select`/`epoll`/`kqueue`) pour **tous** les I/O client↔serveur, **`listen` compris**.
2. `poll()` doit surveiller **lecture ET écriture** simultanément.
3. **Jamais** de `read`/`recv`/`write`/`send` sur une socket/pipe **sans que `poll()` ait dit que le fd est prêt**. Enfreindre ça = **0**.
4. ⚠️ **Interdit de consulter `errno`** après un `read`/`write` pour décider quoi faire. → conséquence de design ci-dessous.
5. ⚠️ **Fichiers disque réguliers = exemptés** : lire/écrire un vrai fichier n'a pas besoin de `poll()` (un fichier est « toujours prêt »). Seuls les fd **qui peuvent attendre** (sockets, pipes/FIFO) sont concernés.

### La conséquence de « errno interdit » (subtil mais essentiel)
Comme tu ne peux pas checker `errno`, ta logique d'I/O devient :
- Tu ne lis/écris que quand `poll()` a signalé le fd prêt.
- Tu fais **un seul** `recv`/`send` (pas de boucle « jusqu'à `EAGAIN` », puisque tu ne peux pas lire `errno`).
- Tu interprètes **la valeur de retour** :
  - `recv` **> 0** → tu as reçu des octets, accumule.
  - `recv` **== 0** → le client a **fermé** la connexion (EOF) → `close()` et nettoie.
  - `recv` **< 0** → traite comme une erreur de connexion → `close()` (sans lire `errno`).
- Pour `send` : la valeur de retour dit combien d'octets sont partis → avance dans ton buffer de sortie ; s'il reste des octets, tu ré-attends `POLLOUT` ; `<= 0` → close.

### Détail poll qui piège tout le monde
Ne surveille `POLLOUT` (écriture) sur un client **que quand tu as réellement quelque chose à lui envoyer**. Sinon `poll()` retourne en boucle « inscriptible » et ton serveur **spinne à 100% CPU**. → tu **actives/désactives** l'intérêt écriture selon que le buffer de sortie est vide ou non.

### Chaque client = une machine à états
À chaque fd client tu attaches une structure. C'est **là** que vit toute la logique :
```
struct Client {
    int          fd;
    std::string  in_buffer;      // octets reçus, pas encore parsés
    ParseState   state;          // REQUEST_LINE / HEADERS / BODY / DONE
    Request      request;        // requête en cours de construction
    std::string  out_buffer;     // réponse à envoyer
    size_t       bytes_sent;     // progression de l'envoi
    bool         want_write;     // faut-il surveiller POLLOUT ?
    CgiState*    cgi;            // état du CGI si route dynamique
    time_t       last_activity;  // pour timeout
};
```
Le serveur n'est rien d'autre qu'un **ensemble de machines à états**, qu'on ne fait avancer **que** quand `poll()` signale le fd correspondant. Une requête peut arriver en **plusieurs morceaux** → plusieurs tours de boucle avant d'être complète. **C'est normal, c'est le point central du projet.**

---

## 3. Sockets & réseau

La séquence TCP pour un socket d'écoute :
```
socket(AF_INET, SOCK_STREAM, 0)        → crée le fd
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, …)  → réutiliser le port sans attendre (évite "address already in use")
bind(fd, host:port)                    → attache le socket à une adresse:port
listen(fd, backlog)                    → passe en mode écoute
fcntl(fd, F_SETFL, O_NONBLOCK)         → non-bloquant
→ ajoute au poll (LECTURE)
```
Quand `poll()` signale un socket d'écoute **lisible**, ça veut dire **« une connexion en attente »** :
```
accept(listen_fd) → new_fd (la connexion client)
fcntl(new_fd, O_NONBLOCK)
→ ajoute new_fd au poll (LECTURE)
```
Tu dois gérer **plusieurs `host:port`** simultanément (plusieurs sockets d'écoute) → « plusieurs sites servis par le même programme ».

---

## 4. La vie d'une requête (bout en bout)

L'histoire complète d'un aller-retour, pour ancrer le modèle :

1. **Connexion.** Le navigateur ouvre une socket TCP vers ton `host:port`. Ton socket d'écoute devient lisible → `accept()` → nouveau fd client dans le poll.
2. **Réception.** `poll` : « client lisible » → `recv()` un bloc → tu l'ajoutes à `in_buffer`.
3. **Parsing incrémental.** Tu tentes de parser : ligne de requête → en-têtes → corps. Si incomplet (`\r\n\r\n` pas encore là, ou corps partiel), tu **attends le prochain tour**. Sinon → requête complète.
4. **Routing.** Tu matches le bon bloc `server` (par `host:port`) puis la bonne `location` (préfixe d'URI le plus long). Méthode autorisée ? Sinon 405.
5. **Traitement.** Selon la méthode et la route :
   - fichier statique (GET) / autoindex,
   - stockage d'un upload (POST),
   - suppression (DELETE),
   - exécution d'un **CGI**,
   - ou **redirection** / **erreur** → code de statut adéquat.
6. **Construction de la réponse.** Tu bâtis `ligne de statut + en-têtes + corps` dans `out_buffer`, tu passes le client en surveillance **écriture**.
7. **Envoi.** `poll` : « client inscriptible » → `send()` par blocs, en avançant `bytes_sent`, jusqu'à tout envoyer.
8. **Fin.** `Connection: keep-alive` → tu **réinitialises** la machine à états du client pour la requête suivante. Sinon `close()`.

---

## 5. Parsing HTTP en détail

Le parsing est **incrémental** : tu accumules dans `in_buffer` au fil des `recv`, et tu avances un **état**.

### Les 3 phases
1. **Ligne de requête** : tu attends un premier `\r\n`. Tu splites en `MÉTHODE`, `URI`, `VERSION`. URI trop longue / malformée → 400.
2. **En-têtes** : chaque ligne `Clé: Valeur` jusqu'à une **ligne vide** (`\r\n\r\n` = fin des en-têtes). Tu stockes dans une map. Clés **insensibles à la casse**.
3. **Corps** : sa présence et sa longueur dépendent des en-têtes ↓.

### Savoir quand le corps est complet — 3 cas
- **`Content-Length: N`** → tu lis exactement **N octets** de corps. Complet quand tu les as tous.
- **`Transfer-Encoding: chunked`** → le corps arrive en morceaux `<taille en hexa>\r\n<données>\r\n`, terminés par un chunk de taille **0**. Tu dois **dé-chunker** (reconstituer le corps réel). Complet au chunk 0.
- **Ni l'un ni l'autre** (typiquement GET) → **pas de corps**.

### Contrôles à faire
- Corps > `client_max_body_size` → **413** (arrête de lire, réponds).
- Requête franchement malformée → **400**.
- Ne **jamais** supposer qu'une requête tient dans un seul `recv`. Ni qu'un `recv` contient une requête entière (il peut en contenir 1,5 en keep-alive → garde le surplus pour la requête suivante).

---

## 6. Le fichier de configuration

Tu t'inspires de la section `server` de NGINX. Le serveur prend **un fichier de conf en argument** (ou un chemin par défaut).

### Le modèle
- Plusieurs blocs **`server`** (chacun = un site sur un `host:port`).
- Dans chaque `server`, plusieurs blocs **`location`** (= des règles par route/URI).
- Sélection : le bon `server` par **`host:port`**, puis la bonne `location` par **préfixe d'URI le plus long**. **Pas de regex exigée.**

### Ce que la conf doit permettre (exigé par le sujet)
| Directive | Rôle |
|---|---|
| `listen` (host:port) | définir **toutes** les paires interface:port d'écoute |
| `error_page` | pages d'erreur custom par code |
| `client_max_body_size` | taille max du corps client (→ 413) |
| **bloc `location`** ↓ | règles par route |
| — méthodes autorisées | liste des méthodes acceptées sur la route (→ 405) |
| — redirection HTTP | renvoyer une 3xx + `Location` |
| — `root` | où trouver le fichier (voir exemple ↓) |
| — directory listing on/off | activer/désactiver l'**autoindex** |
| — fichier par défaut | quel fichier servir quand l'URI est un dossier (**index**) |
| — upload | autoriser l'upload + **où stocker** les fichiers reçus |
| — CGI | exécuter un CGI selon l'**extension** (`.php`, `.py`…) |
| `server_name` | ⚠️ **virtual hosts = HORS SCOPE** — optionnel, seulement si tu veux |

### L'exemple de `root` (tiré du sujet, à connaître)
Si l'URL `/kapouet` est *rootée* sur `/tmp/www`, alors une requête sur `/kapouet/pouic/toto/pouet` cherche le fichier `/tmp/www/pouic/toto/pouet`. → tu **remplaces le préfixe matché** par le `root`, tu ne concatènes pas bêtement.

Tu dois **fournir des fichiers de conf + des fichiers de test** pour démontrer chaque feature en éval.

---

## 7. Routing & méthodes

**Parser la requête → choisir `server` (par host:port) → choisir `location` (préfixe le plus long) → dispatcher par méthode.** Toujours renvoyer le **bon code de statut**.

- **GET** — servir un fichier statique. Si l'URI pointe un **dossier** : servir le fichier `index` configuré, sinon **autoindex** (générer une page HTML listant le dossier via `opendir`/`readdir`) si activé, sinon **403/404**.
- **POST** — recevoir un corps, notamment un **upload de fichier** (stocker à l'emplacement configuré). Gérer `Content-Length` et chunked. → 201/200.
- **DELETE** — supprimer la ressource ciblée. → 200/204, ou 404/403.
- **Méthode non autorisée** sur la route → **405** (+ en-tête `Allow` idéalement).
- **Redirection** configurée → **301/302** + `Location`.
- **Erreurs** → toujours une page d'erreur (custom via `error_page`, sinon **page par défaut** générée par toi — le sujet l'exige si aucune n'est fournie).

---

## 8. CGI — le contenu dynamique

### C'est quoi
Un **CGI** (Common Gateway Interface) laisse ton serveur déléguer une requête à un **programme externe** (php-cgi, python…). Le serveur lance l'interpréteur, lui **passe la requête**, récupère sa **sortie**, et la renvoie comme réponse HTTP. C'est ce qui rend le contenu **dynamique**.

### Le mécanisme
```
1. deux pipes : un pour l'entrée (corps → stdin du CGI), un pour la sortie (stdout du CGI → toi)
2. fork()
3. dans l'enfant :
     dup2() les pipes sur STDIN / STDOUT
     chdir() dans le répertoire du script   ← pour ses chemins relatifs
     execve(interpréteur, [script], env_CGI)
4. dans le parent :
     écris le corps de la requête dans le pipe d'entrée
     lis la sortie du CGI dans le pipe de sortie
     waitpid() pour récupérer l'enfant (éviter les zombies)
```

### Les variables d'environnement CGI
Tu passes la requête au CGI **via l'environnement** (+ le corps via stdin). Les principales :
`REQUEST_METHOD`, `QUERY_STRING`, `CONTENT_LENGTH`, `CONTENT_TYPE`, `PATH_INFO`, `SCRIPT_NAME` / `SCRIPT_FILENAME`, `SERVER_PROTOCOL`, `GATEWAY_INTERFACE`, `SERVER_NAME`, `SERVER_PORT`, `REMOTE_ADDR`…
> Le sujet insiste : **tout le contenu de la requête + les arguments** doivent être disponibles pour le CGI.

### ⚠️ Les précisions explicites du sujet
- **Requêtes chunked** : ton serveur doit **dé-chunker** avant de transmettre → le CGI attend un **EOF** (fermeture du pipe stdin) comme fin de corps.
- **Sortie du CGI** : si le CGI **ne renvoie pas** de `Content-Length`, c'est l'**EOF** (fermeture de son stdout) qui marque la fin des données. Tu lis jusqu'à EOF.
- Le CGI doit tourner **dans le bon répertoire** (`chdir`) pour l'accès par chemin relatif. → c'est pour ça que `chdir` est dans les fonctions autorisées.
- **Au moins un** CGI supporté (php-cgi, Python, peu importe).
- Le CGI produit **ses propres en-têtes** (`Content-Type`, éventuellement `Status`) suivis d'une ligne vide puis du corps → tu **parses** ça et l'intègres dans ta réponse HTTP.

### La subtilité event-driven (piège)
Le CGI peut être lent ou produire beaucoup de données. Si tu `read()` sa sortie de façon **bloquante**, tu gèles tout le serveur. → les **pipes du CGI sont aussi des fd** : idéalement non-bloquants et **surveillés par le poll** comme le reste, pour ne jamais bloquer en attendant le CGI. (Beaucoup d'implémentations gèrent le CGI comme un « client interne » de plus dans la boucle.)

---

## 9. Robustesse

**Non négociable — un crash = note 0 sur tout le projet.**

- Le serveur **ne doit JAMAIS crash ni figer**, quoi qu'on lui envoie — **même à court de mémoire**.
- Traite **toute entrée comme hostile** : requêtes partielles, octets pourris, en-têtes absurdes, corps géants, clients qui coupent en plein milieu, spam de connexions.
- Chaque cas → soit une **réponse d'erreur propre**, soit un **nettoyage de la connexion**. Jamais une exception non rattrapée, jamais un blocage.
- **Une requête ne doit jamais hang indéfiniment** → prévois des **timeouts** (client trop lent = on ferme).
- **Zéro leak, zéro fd qui fuit** : chaque `close()`, chaque `waitpid()`, chaque désallocation au bon endroit.
- L'éval **stress-teste méchamment** (siege, curl tordus, ton propre testeur) et **compare à NGINX** (en-têtes, comportements — attention aux différences de version HTTP).
- Le sujet fournit **un petit testeur** (optionnel) et recommande d'**écrire tes propres tests** (Python, Go, C…). **Ne teste pas qu'avec un seul outil.**

Mantra : **« ton serveur ne meurt jamais »**.

---

## 10. Contraintes & règles de compliance

### Langage & build
- **C++98** strict — doit compiler avec `-std=c++98`.
- Flags obligatoires : **`-Wall -Wextra -Werror`**.
- Privilégie les versions **C++** des fonctions C (`<cstring>` plutôt que `<string.h>`).
- **Aucune bibliothèque externe, ni Boost.**
- **`Makefile`** avec au moins les règles : `$(NAME)`, `all`, `clean`, `fclean`, `re` — **sans relink inutile**.
- Exécution : `./webserv [fichier_de_conf]`.

### Interdits (chacun = fail)
- ❌ Consulter **`errno`** après un `read`/`write`.
- ❌ `read`/`recv`/`write`/`send` sur socket/pipe **sans readiness** du poll.
- ❌ **Plus d'un** `poll()`/mécanisme de multiplexing.
- ❌ **`execve` un autre serveur web**.
- ❌ **`fork`** pour autre chose que le **CGI**.
- ❌ **Crash**, hang, ou fuite.

### Fonctions autorisées (liste exacte du sujet)
`execve, pipe, strerror, gai_strerror, errno, dup, dup2, fork, socketpair, htons, htonl, ntohs, ntohl, select, poll, epoll (epoll_create, epoll_ctl, epoll_wait), kqueue (kqueue, kevent), socket, accept, listen, send, recv, chdir, bind, connect, getaddrinfo, freeaddrinfo, setsockopt, getsockname, getprotobyname, fcntl, close, read, write, waitpid, kill, signal, access, stat, open, opendir, readdir, closedir`.

### Note macOS
Vu que macOS gère `write()` différemment, `fcntl()` y est autorisé **uniquement** avec `F_SETFL`, `O_NONBLOCK`, `FD_CLOEXEC`. Tout autre flag est interdit. (Sur Linux/WSL c'est le même usage pour passer non-bloquant.)

---

## 11. Livrables annexes (c'est noté)

### README.md (obligatoire, racine du repo, en anglais)
- **Première ligne en italique, texte imposé :**
  *This project has been created as part of the 42 curriculum by <login1>[, <login2>…].*
- Section **Description** (but + aperçu).
- Section **Instructions** (compilation / installation / exécution).
- Section **Resources** (références classiques **+ une description de comment l'IA a été utilisée** : pour quelles tâches, quelles parties).

### Chapitre « AI Instructions » (nouveau)
Le sujet a tout un chapitre sur l'IA : sers-t'en pour **comprendre et déblayer**, **jamais** pour du code que tu ne peux pas expliquer. En éval tu dois **justifier chaque partie**, et la **revue par les pairs** est explicitement encouragée. *(D'où ces fiches conceptuelles plutôt que du code clé en main.)*

### Défense
Une **petite modification en direct** peut t'être demandée (changer un comportement, ajouter une mini-feature, adapter une structure de données) — pour vérifier que tu **comprends vraiment** ton code. → structures claires, code lisible.

### Bonus (évalué seulement si le mandatory est parfait)
- **Cookies + gestion de session** (avec exemples simples).
- **Plusieurs types de CGI.**

---

## 12. Répartition binôme & structures partagées

Découpage propre **par couches** :
- **L'un — le cœur réseau** : sockets + boucle `poll()` + machine à états des connexions (le squelette event-driven). **Le nerf du projet, à faire tôt et solide** — tout le reste s'y branche.
- **L'autre — la matière HTTP** : parsing du fichier de conf + parsing des requêtes + construction des réponses.
- **À deux** : routing, méthodes (GET/POST/DELETE), CGI, gestion des erreurs.

⚠️ **Le vrai piège des binômes :** ne pas s'accorder assez tôt sur les **structures partagées**. Mettez-vous d'accord **avant de coder chacun de votre côté** sur :
- la structure **`Request`** (méthode, URI, query, en-têtes, corps, état de parsing),
- la structure **`Response`** (code, en-têtes, corps),
- la **conf parsée** (serveurs → locations → directives),
- l'interface entre « le cœur réseau » et « le HTTP » (qui appelle qui, avec quoi).

Sinon vous recollez deux moitiés qui ne parlent pas la même langue.

---

## 13. Ordre d'attaque

Construis en **couches qui tiennent debout à chaque étape** :

1. **Un `poll()` qui accepte une connexion et renvoie une réponse HTTP en dur** (« Hello »). → la boucle event-driven qui ne bloque pas. **C'est la fondation.**
2. **Parsing requête + construction réponse propres** → servir un **GET statique** réel.
3. **Parsing du fichier de conf** → brancher les vrais ports / routes / `root`.
4. **POST (upload) + DELETE.**
5. **CGI.**
6. **Durcissement** : codes d'erreur exacts, `client_max_body_size`, autoindex, redirections, `error_page`, keep-alive, timeouts, edge cases, **stress-test vs NGINX**.

À chaque palier, ton serveur **marche** (juste avec moins de features). Tu ne construis jamais 3 semaines à l'aveugle.

---

## 14. Checklist des pièges à 0

Les trucs à **graver dès le squelette** — ce sont les fails bêtes qui annulent tout :

- [ ] **Un seul `poll()`**, surveillant **lecture ET écriture**, **`listen` compris**.
- [ ] **Jamais** de `recv`/`send` sur socket/pipe **sans** que poll l'ait dit prêt.
- [ ] **Jamais** consulter **`errno`** après `read`/`write` (déduis tout de la valeur de retour).
- [ ] `recv == 0` → client fermé → `close`. `recv < 0` → `close` (pas d'errno).
- [ ] **N'active `POLLOUT`** que quand tu as des données à envoyer (sinon spin 100% CPU).
- [ ] **`fork` seulement pour le CGI** ; **jamais `execve` un autre serveur web**.
- [ ] Le serveur **ne crash JAMAIS**, même à court de mémoire, même sur entrée pourrie.
- [ ] Une requête **peut arriver en plusieurs morceaux** — ton parsing est incrémental.
- [ ] `Content-Length` de tes réponses **exact**.
- [ ] Chunked : **dé-chunker** en entrée ; CGI attend **EOF**.
- [ ] Pages d'erreur **par défaut** si la conf n'en fournit pas.
- [ ] **Ne teste pas qu'avec un navigateur** : `telnet`, `curl` tordu, tes scripts, comparaison **NGINX**.

---

*Fiche conceptuelle exhaustive de Webserv, alignée sur le sujet officiel **v24.0**. Voir aussi [fiche_webserv_fonctions.md](fiche_webserv_fonctions.md) : les fonctions autorisées, une par une.*
