# Webserv — Les fonctions autorisées, une par une

*Référence des fonctions listées par le sujet officiel **v24.0** (les **seules** autorisées, hors libc de base). Pour chacune : signature, rôle, fonctionnement, limites & pièges Webserv. Groupées par usage plutôt que dans l'ordre du sujet.*

> **Astuce de lecture :** cette liste **est** la carte du projet. Sockets (serveur) + **un** multiplexeur (`poll`) + `fork`/`execve`/`pipe` (CGI) + `open`/`stat`/`opendir` (fichiers statiques). Si tu vois à quoi sert chaque outil, tu vois l'architecture.

---

## Carte mentale

| Bloc | Fonctions | Sert à |
|---|---|---|
| **A. Sockets serveur** | `socket, setsockopt, bind, listen, accept, connect, getsockname, getprotobyname, socketpair` | créer/gérer les connexions TCP |
| **B. Résolution d'adresse** | `getaddrinfo, freeaddrinfo, gai_strerror` | traduire host:port ↔ struct adresse |
| **C. Ordre des octets** | `htons, htonl, ntohs, ntohl` | endianness réseau ↔ machine |
| **D. I/O de données** | `recv, send, read, write, close` | lire/écrire des octets |
| **E. Multiplexing** | `poll, select, epoll_*, kqueue/kevent` | surveiller N fd à la fois (**le cœur**) |
| **F. Contrôle de fd** | `fcntl, dup, dup2, pipe` | non-bloquant, redirection, tuyaux |
| **G. Processus (CGI)** | `fork, execve, waitpid, chdir, kill, signal` | lancer et gérer les CGI |
| **H. Fichiers & dossiers** | `open, stat, access, opendir, readdir, closedir` | servir le statique + autoindex |
| **I. Erreurs** | `errno, strerror` | diagnostiquer (avec ⚠️ une grosse restriction) |

---

## A. Sockets serveur

### `socket`
`int socket(int domain, int type, int protocol)` · `<sys/socket.h>`
- **Rôle :** créer un point de communication (une socket) → renvoie un fd.
- **Fonctionnement :** `socket(AF_INET, SOCK_STREAM, 0)` = IPv4 + TCP. Retour = fd, ou `-1`.
- **Webserv :** premier appel pour chaque `host:port` d'écoute. Le fd obtenu passe ensuite par `setsockopt`→`bind`→`listen`.

### `setsockopt`
`int setsockopt(int fd, int level, int optname, const void *val, socklen_t len)` · `<sys/socket.h>`
- **Rôle :** régler des options de socket.
- **Webserv :** **indispensable** avec `SO_REUSEADDR` (`level = SOL_SOCKET`) → permet de re-`bind` le port immédiatement après un redémarrage, sinon tu te manges « Address already in use » pendant la phase TIME_WAIT.

### `bind`
`int bind(int fd, const struct sockaddr *addr, socklen_t len)` · `<sys/socket.h>`
- **Rôle :** attacher la socket à une **adresse:port** local.
- **Fonctionnement :** tu remplis un `struct sockaddr_in` (famille, port en `htons`, adresse en `htonl`), tu le passes ici.
- **Limite :** échoue si le port est déjà pris (d'où `SO_REUSEADDR`) ou < 1024 sans droits root.

### `listen`
`int listen(int fd, int backlog)` · `<sys/socket.h>`
- **Rôle :** passer la socket en **mode écoute** (accepte des connexions entrantes).
- **`backlog` :** taille de la file d'attente des connexions non encore `accept`ées (ex. `SOMAXCONN` ou 128).

### `accept`
`int accept(int fd, struct sockaddr *addr, socklen_t *len)` · `<sys/socket.h>`
- **Rôle :** extraire **une** connexion en attente → renvoie un **nouveau fd** (la connexion client).
- **Webserv :** appelé **uniquement** quand `poll()` signale le socket d'écoute **lisible**. Le nouveau fd doit être passé **non-bloquant** (`fcntl`) puis ajouté au poll.
- **Limite :** ne renvoie qu'**une** connexion par appel (boucle si plusieurs en attente, mais reste dans la logique poll).

### `connect`
`int connect(int fd, const struct sockaddr *addr, socklen_t len)` · `<sys/socket.h>`
- **Rôle :** établir une connexion **sortante** (côté client).
- **Webserv :** en général **inutile** — tu es un serveur, tu reçois des connexions, tu n'en inities pas. Fourni au cas où (proxy…).

### `getsockname`
`int getsockname(int fd, struct sockaddr *addr, socklen_t *len)` · `<sys/socket.h>`
- **Rôle :** connaître l'**adresse locale** à laquelle une socket est liée.
- **Webserv :** utile pour savoir sur quel `host:port` une connexion est arrivée (choisir le bon `server`) et pour remplir `SERVER_PORT`/`SERVER_NAME` du CGI.

### `getprotobyname`
`struct protoent *getprotobyname(const char *name)` · `<netdb.h>`
- **Rôle :** obtenir le numéro d'un protocole par son nom (`getprotobyname("tcp")`).
- **Webserv :** quasi jamais nécessaire — tu passes `0` (ou `IPPROTO_TCP`) à `socket`. Fonction ancienne, non réentrante.

### `socketpair`
`int socketpair(int domain, int type, int protocol, int sv[2])` · `<sys/socket.h>`
- **Rôle :** créer une **paire de sockets connectées** entre elles (canal bidirectionnel local).
- **Webserv :** alternative à `pipe()` pour dialoguer avec le CGI (un seul canal full-duplex au lieu de deux pipes).

---

## B. Résolution d'adresse

### `getaddrinfo`
`int getaddrinfo(const char *node, const char *service, const struct addrinfo *hints, struct addrinfo **res)` · `<netdb.h>`
- **Rôle :** traduire un `host` + `port` (texte) en **`struct sockaddr`** prêts pour `bind`/`connect`. La façon **moderne** (IPv4/IPv6-agnostique) de remplir les adresses.
- **Fonctionnement :** renvoie une **liste chaînée** de résultats. Retour `0` = OK, sinon un **code d'erreur** (⚠️ **pas** `errno` → voir `gai_strerror`).
- **Limite :** à libérer avec `freeaddrinfo`. Tu peux aussi remplir un `sockaddr_in` à la main sans `getaddrinfo` (plus simple pour de l'IPv4 pur).

### `freeaddrinfo`
`void freeaddrinfo(struct addrinfo *res)` · `<netdb.h>`
- **Rôle :** libérer la liste allouée par `getaddrinfo`. **Sinon → fuite mémoire.**

### `gai_strerror`
`const char *gai_strerror(int errcode)` · `<netdb.h>`
- **Rôle :** message lisible pour un **code de retour de `getaddrinfo`** (qui n'utilise pas `errno`).

---

## C. Ordre des octets (endianness)

Le réseau parle **big-endian** (« network byte order »), ta machine souvent **little-endian**. Les ports et adresses dans les structs doivent être convertis.

### `htons` / `htonl`
`uint16_t htons(uint16_t)` · `uint32_t htonl(uint32_t)` · `<arpa/inet.h>`
- **Rôle :** **h**ost **to** **n**etwork, **s**hort (16 bits) / **l**ong (32 bits).
- **Webserv :** `htons(port)` pour le port (16 bits), `htonl(INADDR_ANY)` pour l'adresse (32 bits), au moment de remplir `sockaddr_in`.

### `ntohs` / `ntohl`
`uint16_t ntohs(uint16_t)` · `uint32_t ntohl(uint32_t)` · `<arpa/inet.h>`
- **Rôle :** l'inverse — **n**etwork **to** **h**ost. Pour **relire** un port/adresse reçus (ex. après `getsockname`/`accept`).

---

## D. I/O de données

### `recv`
`ssize_t recv(int fd, void *buf, size_t len, int flags)` · `<sys/socket.h>`
- **Rôle :** lire des octets **depuis une socket**.
- **Fonctionnement :** retour **`> 0`** = octets reçus ; **`0`** = le client a **fermé** (EOF) → `close` ; **`-1`** = erreur → `close`. `flags` = `0` en général.
- **⚠️ Webserv :** appelé **uniquement** après readiness `poll()`. **Un seul** appel (pas de boucle jusqu'à `EAGAIN`). **Interdit de consulter `errno` après.** Tu décides tout via la valeur de retour.

### `send`
`ssize_t send(int fd, const void *buf, size_t len, int flags)` · `<sys/socket.h>`
- **Rôle :** écrire des octets **vers une socket**.
- **Fonctionnement :** retour = **nombre d'octets réellement envoyés**, souvent **< `len`** (envoi **partiel** !) → tu dois mémoriser la progression et renvoyer le reste au prochain `POLLOUT`.
- **⚠️ Piège SIGPIPE :** écrire vers une socket que le pair a fermée envoie un **`SIGPIPE`** qui **tue ton process par défaut** (= crash = 0). → **`signal(SIGPIPE, SIG_IGN)`** au démarrage (ou flag `MSG_NOSIGNAL` sous Linux).
- **⚠️** même règle : que sur readiness poll, **pas d'`errno`** après.

### `read`
`ssize_t read(int fd, void *buf, size_t count)` · `<unistd.h>`
- **Rôle :** lire des octets d'un fd **générique** (fichier, pipe).
- **Webserv :** pour lire un **fichier statique** (fichier = exempt de poll, « toujours prêt ») ou la **sortie d'un CGI** (pipe → à surveiller par poll). Mêmes conventions de retour que `recv` ; **`errno` interdit après**.

### `write`
`ssize_t write(int fd, const void *buf, size_t count)` · `<unistd.h>`
- **Rôle :** écrire des octets vers un fd générique.
- **Webserv :** écrire un **upload** sur disque, ou le **corps de requête dans le stdin du CGI**. Écritures **partielles** possibles (surtout vers un pipe). **`errno` interdit après.**

### `close`
`int close(int fd)` · `<unistd.h>`
- **Rôle :** fermer un fd et le libérer.
- **⚠️ Webserv :** **chaque** fd ouvert (client, pipe CGI, fichier) doit être `close`. Un fd oublié = **fuite de fd** → au bout d'un moment `accept`/`open` échouent (« too many open files »).

---

## E. Multiplexing — le cœur

**Tu n'as le droit qu'à UN seul de ces mécanismes, pour TOUS les I/O.** `poll` est le choix standard et portable.

### `poll` ⭐
`int poll(struct pollfd *fds, nfds_t nfds, int timeout)` · `<poll.h>`
- **Rôle :** surveiller un **tableau de fd** ; bloque jusqu'à ce qu'au moins un soit prêt.
- **Fonctionnement :** `struct pollfd { int fd; short events; short revents; }`. Tu remplis `events` (`POLLIN` = lecture, `POLLOUT` = écriture) ; au retour, `revents` dit ce qui est prêt (+ `POLLHUP`/`POLLERR`). Retour = nb de fd prêts, `0` = timeout, `-1` = erreur. `timeout` en **ms** (`-1` = bloque indéfiniment).
- **⚠️ Webserv :** **un seul** `poll` pour tout, **`listen` compris**, surveillant **lecture ET écriture**. N'active `POLLOUT` que quand tu as des données à envoyer (sinon **spin 100% CPU**). Un `timeout` fini permet aussi de réveiller la boucle pour gérer les **timeouts clients**.

### `select`
`int select(int nfds, fd_set *r, fd_set *w, fd_set *e, struct timeval *timeout)` · `<sys/select.h>`
- **Rôle :** même idée que `poll`, en plus ancien.
- **Limites :** plafonné à **`FD_SETSIZE` (1024)** fd ; il faut **reconstruire les `fd_set`** à chaque appel (macros `FD_ZERO/FD_SET/FD_CLR/FD_ISSET`), et `nfds` = plus grand fd + 1. Plus pénible que `poll`.

### `epoll_create` / `epoll_ctl` / `epoll_wait`
`<sys/epoll.h>` · **Linux uniquement**
- **Rôle :** multiplexeur scalable (O(1) au lieu de O(n)). `epoll_create` crée l'instance, `epoll_ctl` ajoute/modifie/retire un fd, `epoll_wait` attend les événements.
- **Limites :** non portable (pas sur macOS), et le mode **edge-triggered** (`EPOLLET`) impose de **vider** complètement chaque fd → plus complexe. Le **level-triggered** (défaut) ressemble à `poll`. Utile surtout à très grande échelle — overkill pour Webserv, mais autorisé.

### `kqueue` / `kevent`
`<sys/event.h>` · **BSD/macOS uniquement**
- **Rôle :** l'équivalent d'epoll côté BSD/macOS. `kqueue` crée l'instance, `kevent` enregistre les filtres et récupère les événements.
- **Webserv :** à choisir si tu développes sur Mac et veux du scalable. Sinon `poll` fait le taf partout.

---

## F. Contrôle de fd

### `fcntl`
`int fcntl(int fd, int cmd, ...)` · `<fcntl.h>`
- **Rôle :** manipuler les propriétés d'un fd.
- **Webserv :** **`fcntl(fd, F_SETFL, O_NONBLOCK)`** pour rendre chaque socket/pipe **non-bloquant** — obligatoire. `FD_CLOEXEC` (via `F_SETFD`) pour que les fd ne fuitent pas dans le CGI après `execve`.
- **⚠️ macOS :** `fcntl` autorisé **uniquement** avec `F_SETFL`, `O_NONBLOCK`, `FD_CLOEXEC`. Tout autre flag = interdit.

### `dup`
`int dup(int oldfd)` · `<unistd.h>`
- **Rôle :** dupliquer un fd vers le **plus petit fd libre**.
- **Webserv :** peu utilisé directement ; `dup2` est plus précis.

### `dup2`
`int dup2(int oldfd, int newfd)` · `<unistd.h>`
- **Rôle :** dupliquer `oldfd` **sur** `newfd` (ferme `newfd` d'abord si besoin).
- **Webserv :** **clé du CGI** — dans l'enfant : `dup2(pipe_in, STDIN_FILENO)` et `dup2(pipe_out, STDOUT_FILENO)` pour que le CGI lise la requête sur son stdin et écrive sur son stdout.

### `pipe`
`int pipe(int pipefd[2])` · `<unistd.h>`
- **Rôle :** créer un tuyau **unidirectionnel** : `pipefd[0]` = lecture, `pipefd[1]` = écriture.
- **Webserv :** deux pipes pour le CGI (un pour lui envoyer le corps, un pour lire sa sortie). Côtés inutilisés à **fermer** dans chaque process (sinon l'EOF n'arrive jamais → le CGI hang).

---

## G. Processus — le CGI

### `fork`
`pid_t fork(void)` · `<unistd.h>`
- **Rôle :** dupliquer le process → un **enfant**. Retour : **`0`** dans l'enfant, **PID de l'enfant** dans le parent, `-1` si échec.
- **⚠️ Webserv :** **autorisé uniquement pour le CGI.** L'utiliser ailleurs = fail.

### `execve`
`int execve(const char *path, char *const argv[], char *const envp[])` · `<unistd.h>`
- **Rôle :** **remplacer** l'image du process courant par un autre programme (l'interpréteur CGI). Ne **revient jamais** si ça marche (sinon retourne `-1`).
- **Fonctionnement :** `argv` = `[interpréteur, script, NULL]` ; `envp` = tes **variables CGI**.
- **⚠️ Webserv :** **interdit d'`execve` un autre serveur web.** À faire dans l'enfant, après `dup2` et `chdir`.

### `waitpid`
`pid_t waitpid(pid_t pid, int *status, int options)` · `<sys/wait.h>`
- **Rôle :** attendre/**récupérer** un enfant terminé (évite les **zombies**). `status` renseigne le code de sortie (`WIFEXITED`, `WEXITSTATUS`).
- **⚠️ Webserv :** utilise **`WNOHANG`** pour **ne pas bloquer** la boucle en attendant le CGI — tu vérifies s'il a fini, sinon tu continues à servir les autres clients.

### `chdir`
`int chdir(const char *path)` · `<unistd.h>`
- **Rôle :** changer le **répertoire courant**.
- **Webserv :** dans l'enfant CGI, **avant `execve`**, pour se placer dans le dossier du script → ses **chemins relatifs** fonctionnent. (Explicitement demandé par le sujet.)

### `kill`
`int kill(pid_t pid, int sig)` · `<signal.h>`
- **Rôle :** envoyer un **signal** à un process.
- **Webserv :** pour **tuer un CGI** qui traîne / dépasse un timeout : `kill(pid, SIGKILL)`. Robustesse.

### `signal`
`void (*signal(int signum, void (*handler)(int)))(int)` · `<signal.h>`
- **Rôle :** définir le **comportement** face à un signal.
- **⚠️ Webserv (crucial) :** **`signal(SIGPIPE, SIG_IGN)`** au démarrage — sinon un `send` vers un client parti **tue le serveur** (crash = 0). Éventuellement gérer `SIGINT` pour un arrêt propre.

---

## H. Fichiers & dossiers (statique + autoindex)

### `open`
`int open(const char *path, int flags, ... mode)` · `<fcntl.h>`
- **Rôle :** ouvrir un fichier → fd.
- **Fonctionnement :** `flags` : `O_RDONLY` (servir un fichier), `O_WRONLY | O_CREAT | O_TRUNC` (écrire un upload), `mode` = permissions si création.
- **Webserv :** un fichier régulier est **exempt du poll** (« toujours prêt ») → tu peux le `read`/`write` directement.

### `stat`
`int stat(const char *path, struct stat *buf)` · `<sys/stat.h>`
- **Rôle :** récupérer les **métadonnées** d'un chemin.
- **Webserv :** `buf.st_size` → le **`Content-Length`** ; `S_ISDIR(buf.st_mode)` → est-ce un **dossier** (→ index/autoindex) ; `S_ISREG` → fichier régulier ; permissions → 403. **Très utilisé** dans le service statique.

### `access`
`int access(const char *path, int mode)` · `<unistd.h>`
- **Rôle :** tester **existence/permissions** d'un chemin (`F_OK` existe, `R_OK` lisible, `W_OK`, `X_OK` exécutable).
- **Webserv :** check rapide d'existence (→ 404) ou d'exécutabilité d'un script CGI. (⚠️ `stat` est souvent préférable — `access` a des subtilités de droits réels/effectifs.)

### `opendir`
`DIR *opendir(const char *name)` · `<dirent.h>`
- **Rôle :** ouvrir un **flux de répertoire**. Renvoie `NULL` si échec.

### `readdir`
`struct dirent *readdir(DIR *dirp)` · `<dirent.h>`
- **Rôle :** lire les **entrées une par une** (`d_name` = nom). Renvoie `NULL` en fin de dossier.
- **Webserv :** cœur de l'**autoindex** — tu boucles pour lister le contenu du dossier et générer la page HTML.

### `closedir`
`int closedir(DIR *dirp)` · `<dirent.h>`
- **Rôle :** fermer le flux. **À ne pas oublier** (fuite sinon).

---

## I. Erreurs

### `errno`
`extern int errno` · `<errno.h>`
- **Rôle :** variable (thread-local) positionnée par les syscalls en cas d'échec (`EAGAIN`, `EADDRINUSE`, `EPIPE`…).
- **⚠️ Webserv — la grosse restriction :** *« consulter `errno` pour ajuster le comportement du serveur est strictement interdit **après un `read`/`write`** »* (donc aussi après `recv`/`send`) → **note 0**. Tu déduis l'état **uniquement** de la valeur de retour + de ce que `poll()` a dit. Pour les appels de **setup** (`socket`/`bind`/`listen`), l'inspecter pour un message d'erreur au lancement reste tolérable — mais **jamais** dans le chemin d'I/O.

### `strerror`
`char *strerror(int errnum)` · `<cstring>`
- **Rôle :** message lisible pour une valeur d'`errno` (`strerror(errno)`).
- **Webserv :** pratique pour logger une erreur **de setup** (« bind failed: … »). Mais comme tu ne peux pas lire `errno` après un I/O, tu ne l'utilises **pas** dans la boucle de lecture/écriture.

---

## Récap : le minimum vital vs le confort

- **Indispensables** : `socket, setsockopt, bind, listen, accept, poll, fcntl, recv, send, close, read, write, open, stat, fork, execve, pipe, dup2, waitpid, chdir, signal, opendir, readdir, closedir`.
- **Confort / selon choix** : `getaddrinfo`/`freeaddrinfo`/`gai_strerror` (vs remplir `sockaddr_in` à la main), `getsockname`, `socketpair` (vs 2 `pipe`), `kill`, `access`.
- **Rarement nécessaires** : `connect`, `getprotobyname`, `dup`, `select`/`epoll`/`kqueue` (si tu prends `poll`).

---

*Fiche des fonctions autorisées, basée sur la liste exacte du sujet **Webserv v24.0**. Voir aussi [fiche_webserv_conceptuelle.md](fiche_webserv_conceptuelle.md) : le projet de bout en bout. Signatures = prototypes POSIX standard ; voir les `man` correspondants (`man 2 poll`, `man 2 accept`…) pour chaque flag.*
