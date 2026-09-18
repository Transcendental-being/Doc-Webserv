# Webserv — Plan de répartition binôme (tout ce qui reste)

*État de départ : le **parseur de requêtes HTTP** (`RequestParser` + `Request` + `Utils`) est **fini et testé** (parsing incrémental validé, codes 400/413/414/431/501/505 exacts). Reste tout le serveur autour.*

> Convention de nommage : **Dev A = « Cœur réseau »**, **Dev B = « Matière HTTP »**, **À DEUX = les couches qui touchent aux deux**. C'est le découpage recommandé par la fiche conceptuelle (§12).

---

## 0. AVANT de coder chacun de son côté (à deux, 1 session)

Le vrai piège du binôme : recoller deux moitiés qui ne parlent pas la même langue. On **fige les structures partagées d'abord** :

- [ ] `struct Client` — l'interface entre le réseau et le HTTP :
  ```cpp
  struct Client {
      int          fd;
      std::string  out_buffer;    // rempli par B (Response), vidé par A (send)
      size_t       bytes_sent;
      bool         want_write;
      RequestParser parser;       // fourni, appartient à B
      // + état CGI (rempli plus tard, à deux)
      time_t       last_activity;
  };
  ```
- [ ] `Request` (déjà figé — ne pas y toucher sans se prévenir).
- [ ] `Response` (code, headers map, body) — B en est owner, mais A doit connaître son API `toString()`.
- [ ] La **config parsée** : `ServerConfig` → `vector<LocationConfig>`. On fige les champs ensemble.
- [ ] Le **contrat d'appel** : *qui appelle qui ?*
  - A (boucle poll) : `recv` → `client.parser.feed()` → `client.parser.parse()`.
  - Si `PARSE_COMPLETE` → A appelle `handleRequest(client, config)` (couche « à deux ») qui remplit `client.out_buffer` et met `want_write = true`.

**Tant que ces 5 points ne sont pas écrits noir sur blanc, on ne se sépare pas.**

---

## Dev A — Cœur réseau (le squelette event-driven)

C'est **le nerf du projet**, à faire tôt et solide. Tout se branche dessus.

### A1. Sockets d'écoute
- [ ] `socket()` → `setsockopt(SO_REUSEADDR)` → `bind()` → `listen()` → `fcntl(O_NONBLOCK)`
- [ ] Gérer **plusieurs `host:port`** (plusieurs sockets d'écoute).

### A2. La boucle `poll()` (LA règle d'or)
- [ ] **Un seul `poll()`** pour tous les fd, `listen` compris.
- [ ] Surveille **lecture ET écriture** simultanément.
- [ ] **Jamais** de `recv`/`send` sans readiness du poll. **Jamais** `errno` après read/write.
- [ ] `recv == 0` → close ; `recv < 0` → close (déduit de la valeur de retour, pas d'errno).
- [ ] `POLLOUT` **activé seulement** quand `out_buffer` non vide (sinon spin 100 % CPU).

### A3. Cycle de vie client
- [ ] `accept()` → nouveau `Client` non-bloquant ajouté au poll.
- [ ] Lisible → `recv` un bloc → `feed`/`parse` → si complet, appelle la couche traitement.
- [ ] Inscriptible → `send` un bloc → avance `bytes_sent` → tout envoyé ? keep-alive (reset) ou `close()`.
- [ ] **Timeouts** : client trop lent → close (via `last_activity`).
- [ ] Zéro leak, zéro fd qui fuit : chaque `close()` au bon endroit.

### A4. Robustesse réseau
- [ ] `signal(SIGPIPE, SIG_IGN)` pour ne pas mourir sur un send vers socket fermée.
- [ ] Le serveur **ne crash / ne fige JAMAIS**, même sous spam de connexions.

**Livrable A :** un serveur qui accepte N clients simultanés et renvoie une réponse HTTP en dur sans jamais bloquer. → c'est la fondation (étape 1 de l'ordre d'attaque).

---

## Dev B — Matière HTTP (config + réponses)

Le parseur de requêtes est déjà à toi et fini. Il te reste **deux gros morceaux**.

### B1. Classe `Response`
- [ ] `setStatus(code)` avec table code→raison (200 OK, 201, 204, 301/302, 400, 403, 404, 405, 413, 500, 501…).
- [ ] Headers : `Content-Type` (via `Utils::mimeType`), `Content-Length` **exact**, `Date`, `Server`, `Connection`, `Location` (3xx), `Allow` (405).
- [ ] `toString()` → `ligne de statut + headers + \r\n\r\n + body`.
- [ ] **Pages d'erreur par défaut générées** si la config n'en fournit pas (le sujet l'exige).

### B2. Parseur du fichier de configuration (un 2ᵉ parseur complet)
- [ ] Lexer/parser du format type NGINX : blocs `server { ... }` contenant des blocs `location { ... }`.
- [ ] Directives serveur : `listen` (host:port, multiples), `error_page`, `client_max_body_size`, `server_name` (optionnel/hors scope).
- [ ] Directives location : méthodes autorisées, `root`, `index`, autoindex on/off, redirection (3xx + `Location`), upload (on + dossier de stockage), CGI (extension → interpréteur).
- [ ] Structures `ServerConfig` / `LocationConfig` remplies (celles figées en §0).
- [ ] **Validation** : conf invalide → message d'erreur clair + exit propre (jamais de crash).

**Livrable B :** `Response` fonctionnelle + une config d'exemple parsée correctement en mémoire.

---

## À DEUX — les couches qui touchent réseau ET HTTP

À attaquer **une fois A2 (poll) et B1/B2 debout**. On code ces morceaux ensemble ou en pair-programming, car ils lisent la config ET produisent des réponses.

### C1. Routing (`handleRequest`)
- [ ] Match du bon `server` par **host:port** (via header `Host` + socket d'écoute).
- [ ] Match de la bonne `location` par **préfixe d'URI le plus long**.
- [ ] Vérif méthode autorisée → sinon **405** (+ `Allow`).
- [ ] Résolution du chemin disque : **remplacer le préfixe matché par `root`** (ex. `/kapouet` → `/tmp/www`, pas de concat bête).
- [ ] `client_max_body_size` dépassé → **413**.

### C2. Méthodes
- [ ] **GET** : fichier statique ; si dossier → `index` configuré, sinon **autoindex** (`opendir`/`readdir` → HTML), sinon 403/404.
- [ ] **POST** : réception du corps (Content-Length + chunked déjà dé-chunké par le parseur) → **upload** stocké à l'emplacement configuré → 201/200.
- [ ] **DELETE** : suppression de la ressource → 200/204, ou 404/403.
- [ ] **Redirection** configurée → 301/302 + `Location`.

### C3. CGI
- [ ] 2 pipes + `fork` + `dup2` (stdin/stdout) + `chdir` (répertoire du script) + `execve`.
- [ ] Variables d'env : `REQUEST_METHOD`, `QUERY_STRING`, `CONTENT_LENGTH`, `CONTENT_TYPE`, `PATH_INFO`, `SCRIPT_NAME/FILENAME`, `SERVER_PROTOCOL`, `GATEWAY_INTERFACE`, `SERVER_NAME/PORT`, `REMOTE_ADDR`.
- [ ] Corps de la requête → stdin du CGI ; CGI attend **EOF** (fermeture du pipe).
- [ ] Sortie CGI : parser ses propres headers + ligne vide + corps ; si pas de `Content-Length`, lire **jusqu'à EOF**.
- [ ] **Pipes CGI surveillés par le poll** (fd non-bloquants, « client interne ») pour ne jamais bloquer le serveur.
- [ ] `waitpid` pour éviter les zombies. `fork` **uniquement** ici.

### C4. Durcissement final (à deux)
- [ ] Codes d'erreur exacts partout, `error_page` custom appliqués.
- [ ] keep-alive vs `close` selon `Connection` et version HTTP.
- [ ] Timeouts robustes.
- [ ] **Stress-test** : `siege`, `curl` tordus, telnet, scripts perso, **comparaison NGINX**.
- [ ] Zéro leak (valgrind), zéro fd qui fuit.

---

## Livrables annexes (notés — à se répartir)

- [ ] **README.md** (racine, anglais) : 1ʳᵉ ligne en italique imposée, Description, Instructions, Resources (**+ comment l'IA a été utilisée**). → **Dev ___**
- [ ] **Makefile** final : `all clean fclean re $(NAME)`, sans relink inutile, `-Wall -Wextra -Werror -std=c++98`. → **Dev A**
- [ ] **Fichiers de conf + fichiers de test** démontrant chaque feature en éval. → **à deux**
- [ ] Chacun doit **savoir justifier tout le code** (défense + petite modif en direct possible).

---

## Ordre d'exécution conseillé (jalons communs)

| Jalon | A (réseau) | B (HTTP) | À deux |
|---|---|---|---|
| **J1** | poll + accept + réponse en dur | `Response` de base | §0 structures figées |
| **J2** | recv/send + machine à états client | parseur de conf | brancher parser → routing → GET statique |
| **J3** | timeouts + keep-alive | pages d'erreur + error_page | POST (upload) + DELETE |
| **J4** | pipes CGI dans le poll | env CGI + parse sortie CGI | CGI complet |
| **J5** | — | — | durcissement + stress-test vs NGINX |

À chaque jalon le serveur **marche** (avec moins de features). On ne construit jamais à l'aveugle.

---

## Les fails à 0 à graver dès le squelette (responsabilité A surtout)

- [ ] Un seul `poll()`, lecture **ET** écriture, `listen` compris.
- [ ] Jamais `recv`/`send` sans readiness. Jamais `errno` après read/write.
- [ ] `recv == 0` → close ; `recv < 0` → close.
- [ ] `POLLOUT` seulement si données à envoyer.
- [ ] `fork` **seulement** pour le CGI ; jamais `execve` un autre serveur web.
- [ ] Ne crash JAMAIS, même OOM, même entrée pourrie.
- [ ] `Content-Length` des réponses **exact**.

---

*Basé sur `fiche_webserv_conceptuelle.md` (Webserv v24.0). Le parseur de requêtes étant déjà livré, le chemin critique est maintenant : **A2 (poll) → C1 (routing) → C2 (GET) → reste**.*
