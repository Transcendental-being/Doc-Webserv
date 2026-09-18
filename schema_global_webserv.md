# Webserv de A à Z — le fonctionnement global

*Pour comprendre la machine entière, pas juste ta pièce. Aucun code : le déroulé, dans l'ordre, avec le **pourquoi** de chaque étape.*

> **Le programme en une phrase :** `./webserv conf.conf` lit une config, ouvre des sockets d'écoute, puis tourne **à l'infini** dans une boucle qui surveille tous les descripteurs de fichier et fait avancer chaque client d'un petit pas à chaque tour.

---

## Le schéma global

```
  ./webserv conf.conf
         │
         ▼
╔════════════════════════════════════════════════════════════════════╗
║  PHASE 0 — DÉMARRAGE (une seule fois)                              ║
║                                                                     ║
║   ConfigParser lit conf.conf  ──►  vector<ServerConfig>            ║
║                                          │                          ║
║   pour chaque host:port à écouter :      ▼                          ║
║      socket() → setsockopt() → bind() → listen() → non-bloquant    ║
║      → on ajoute ce fd à la liste surveillée par poll (LECTURE)    ║
╚════════════════════════════════════════════════════════════════════╝
         │
         ▼
╔════════════════════════════════════════════════════════════════════╗
║  BOUCLE INFINIE  ──  un seul poll() pour TOUT                      ║
║                                                                     ║
║   poll(tous les fd)   ← BLOQUE ici jusqu'à ce qu'au moins un fd    ║
║         │               soit prêt (lisible ou inscriptible)        ║
║         ▼                                                           ║
║   pour chaque fd prêt, 3 cas possibles :                           ║
║                                                                     ║
║   ┌──── CAS A ─────────┐ ┌──── CAS B ─────────┐ ┌─ CAS C ────────┐ ║
║   │ socket d'ÉCOUTE    │ │ client LISIBLE     │ │ client         │ ║
║   │ est lisible        │ │                    │ │ INSCRIPTIBLE   │ ║
║   │ = qqn se connecte  │ │ = des octets       │ │ = on peut lui  │ ║
║   │                    │ │   sont arrivés     │ │   envoyer      │ ║
║   │ accept()           │ │                    │ │                │ ║
║   │ → nouveau fd       │ │ recv() UN bloc     │ │ send() UN bloc │ ║
║   │ → non-bloquant     │ │        │           │ │ du out_buffer  │ ║
║   │ → ajoute au poll   │ │        ▼           │ │        │       │ ║
║   │ → crée un Client   │ │   [PHASES 1→5]     │ │        ▼       │ ║
║   └────────────────────┘ └────────────────────┘ │  tout envoyé ? │ ║
║                                                  │  ├ oui → keep- │ ║
║                                                  │  │  alive ou   │ ║
║                                                  │  │  close()    │ ║
║                                                  │  └ non → on    │ ║
║                                                  │     reboucle   │ ║
║                                                  └────────────────┘ ║
║         │                                                           ║
║         └──────────── on reboucle ────────────────────────────────►║
╚════════════════════════════════════════════════════════════════════╝
```

---

## Le détail du CAS B : que se passe-t-il quand des octets arrivent

C'est là que vit **tout** le travail HTTP. Voici le chemin complet, de l'octet reçu à la réponse prête.

```
recv() a mis des octets dans in_buffer
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ PHASE 1 — PARSING          [RequestParser — DÉJÀ FINI]  │
│                                                          │
│   parser.feed(octets) puis parser.parse()               │
│                                                          │
│   3 verdicts possibles :                                │
│                                                          │
│   INCOMPLETE ──► on ne fait RIEN, on attend le prochain │
│                   tour de boucle (il manque des octets) │
│                   ⚠ C'est NORMAL et c'est le cœur du    │
│                     projet : une requête arrive souvent │
│                     en plusieurs morceaux.              │
│                                                          │
│   ERROR ──────► le parseur donne un code (400/414/431/  │
│                  501/505) → saute direct à la PHASE 4   │
│                                                          │
│   COMPLETE ───► on a un objet Request propre → PHASE 2  │
└─────────────────────────────────────────────────────────┘
         │ (COMPLETE)
         ▼
┌─────────────────────────────────────────────────────────┐
│ PHASE 2 — ROUTING                        [à deux]       │
│                                                          │
│  a) quel bloc `server` ? → par le host:port de la       │
│     socket d'écoute qui a reçu la connexion             │
│                                                          │
│  b) quelle `location` ? → le PRÉFIXE D'URI LE PLUS LONG │
│     qui matche (ex: /images bat / pour /images/a.png)   │
│                                                          │
│  c) la méthode est-elle autorisée sur cette route ?     │
│     non → code 405 → PHASE 4                            │
│                                                          │
│  d) une redirection est configurée ? → 301/302 → PHASE 4│
│                                                          │
│  e) traduire l'URI en CHEMIN DISQUE :                   │
│     on remplace le préfixe matché par le `root`         │
│     ex: /kapouet rooté sur /tmp/www                     │
│         → /kapouet/pouic/toto  devient                  │
│           /tmp/www/pouic/toto                           │
│     (on REMPLACE le préfixe, on ne concatène pas)       │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ PHASE 3 — TRAITEMENT (selon la méthode)   [à deux]      │
│                                                          │
│  GET    → le chemin existe ?                            │
│           • fichier    → on le lit         → 200        │
│           • dossier    → fichier `index` configuré ?    │
│                          sinon autoindex activé ?       │
│                          → on génère le HTML du listing │
│                          sinon                → 403     │
│           • rien       →                      → 404     │
│           • pas les droits →                  → 403     │
│                                                          │
│  POST   → upload : on écrit le corps reçu dans le       │
│           dossier configuré              → 201          │
│           (le corps est déjà dé-chunké par le parseur)  │
│                                                          │
│  DELETE → on supprime le fichier         → 204          │
│           (ou 404 / 403)                                │
│                                                          │
│  CGI    → si l'extension matche (.py, .php) :           │
│           pipes + fork + dup2 + chdir + execve          │
│           on écrit le corps dans son stdin              │
│           on lit sa sortie (ses propres en-têtes + corps)│
│           ⚠ les pipes sont des fd → ils vont AUSSI      │
│             dans le poll (sinon on gèle le serveur)     │
│           si ça casse                    → 500          │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ PHASE 4 — CONSTRUCTION DE LA RÉPONSE     [TOI, Dev B]   │
│                                                          │
│  On arrive ici avec :                                   │
│    • un CODE (du parseur, du routing, ou du traitement) │
│    • un CORPS (contenu du fichier, page générée,        │
│                page d'erreur, sortie CGI)               │
│                                                          │
│  La Response fabrique :                                 │
│    ligne de statut + en-têtes + ligne vide + corps      │
│    avec Content-Length EXACT, Content-Type, Date...     │
│                                                          │
│  toString() → une grande string                         │
└─────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│ PHASE 5 — ARMER L'ENVOI                  [Dev A]        │
│                                                          │
│  out_buffer = la string produite                        │
│  on ACTIVE la surveillance ÉCRITURE (POLLOUT) sur ce fd │
│                                                          │
│  ⚠ On n'envoie PAS tout de suite ! On dit juste à poll  │
│    « préviens-moi quand ce client pourra recevoir ».    │
│    → le send() se fera au CAS C, à un prochain tour.    │
└─────────────────────────────────────────────────────────┘
```

---

## Pourquoi cette architecture bizarre (le « pourquoi » qui débloque tout)

### Le problème à résoudre
**Un seul thread** doit servir **plusieurs clients à la fois**. Si tu fais un `recv()` bloquant sur un client lent (ou malveillant, qui envoie 1 octet par minute), **tout le serveur gèle** en l'attendant. Inacceptable.

### La solution
Ne jamais attendre. Tu ne touches à un fd **que** quand `poll()` t'a garanti qu'il est prêt, et tu ne fais **qu'un petit pas** à chaque fois :
- un `recv`, pas une boucle jusqu'à tout avoir,
- un `send`, pas une boucle jusqu'à tout envoyer.

Puis tu **reboucles**. Ainsi 100 clients progressent « en même temps » sur un seul thread, sans qu'aucun ne bloque les autres.

### La conséquence : chaque client est une machine à états
Comme tu avances par petits pas, tu dois **mémoriser où tu en es** pour chaque client entre deux tours de boucle. D'où une structure par client :

```
struct Client {
    fd              le descripteur
    in_buffer       octets reçus pas encore parsés
    parser          l'état du parsing (REQUEST_LINE / HEADERS / BODY...)
    out_buffer      la réponse à envoyer
    bytes_sent      combien on a déjà envoyé
    want_write      faut-il surveiller POLLOUT ?
    last_activity   pour le timeout
}
```

**C'est pour ça que ton `RequestParser` est incrémental.** Tu l'avais codé sans savoir pourquoi — voilà le pourquoi : il est rappelé à chaque tour de boucle avec quelques octets de plus, et il doit reprendre exactement là où il s'était arrêté.

---

## Le cycle de vie complet d'une connexion

```
1. CONNEXION     le navigateur ouvre une socket TCP
                 → ta socket d'écoute devient lisible
                 → accept() → nouveau fd + struct Client

2. RÉCEPTION     poll dit « lisible » → recv() un bloc
   (répété)      → feed au parser → INCOMPLETE → on attend
                 → poll dit « lisible » → recv() encore...
                 → ... jusqu'à COMPLETE

3. TRAITEMENT    routing → méthode → on obtient code + corps

4. RÉPONSE       Response.toString() → out_buffer
                 → on active POLLOUT

5. ENVOI         poll dit « inscriptible » → send() un bloc
   (répété)      → reste des octets ? on attend le prochain tour
                 → ... jusqu'à tout envoyé

6. FIN           Connection: keep-alive ?
                 ├ OUI  → on RÉINITIALISE la machine à états
                 │        (⚠ en gardant le surplus du in_buffer :
                 │         un recv peut contenir 1,5 requête !)
                 │        → retour à l'étape 2
                 └ NON  → close() et nettoyage
```

---

## Qui fait quoi

| Composant | Rôle | Qui |
|---|---|---|
| **ConfigParser** | fichier .conf → structures en mémoire | **Toi (B)** |
| **Sockets + poll + boucle** | le squelette event-driven | **Dev A** |
| **Client (machine à états)** | mémoriser où en est chaque connexion | **Dev A** |
| **RequestParser** | octets → objet Request | ✅ **fait** |
| **Routing** | choisir server/location, traduire l'URI | **à deux** |
| **Méthodes** (GET/POST/DELETE) | produire le contenu | **à deux** |
| **CGI** | déléguer à un programme externe | **à deux** |
| **Response** | infos → octets HTTP valides | **Toi (B)** |

**Tes deux pièces (ConfigParser et Response) sont aux deux extrémités :** l'une alimente le serveur au démarrage, l'autre emballe la sortie. Entre les deux, c'est le réseau de Dev A et le routing commun.

---

## Les règles d'or (chacune = 0 si violée)

1. **Un seul `poll()`** pour tous les I/O, **socket d'écoute comprise**.
2. `poll()` surveille **lecture ET écriture** simultanément.
3. **Jamais** de `recv`/`send` sur socket ou pipe **sans que `poll()` ait dit prêt**.
4. **Jamais** consulter **`errno`** après un `read`/`write` → tout se déduit de la valeur de retour :
   - `recv > 0` → des octets, on accumule
   - `recv == 0` → le client a fermé → `close()`
   - `recv < 0` → erreur → `close()`
5. **N'active `POLLOUT` que quand il y a des données à envoyer** (sinon `poll` retourne en boucle « inscriptible » → 100 % CPU).
6. **`fork` uniquement pour le CGI.** Jamais `execve` un autre serveur web.
7. **Le serveur ne crash JAMAIS**, quoi qu'on lui envoie. Toute entrée est hostile.
8. **Timeouts** : un client trop lent, on le ferme (sinon il squatte un fd à vie).
9. Les **fichiers disque réguliers sont exemptés** du poll (un fichier est toujours « prêt »). Seuls les fd qui peuvent *attendre* (sockets, pipes) sont concernés.

---

## L'ordre de construction (pour que ça tienne debout à chaque palier)

1. **poll + accept + une réponse en dur** (« Hello ») → la fondation event-driven.
2. **Parsing + Response** branchés → servir un vrai GET statique.
3. **ConfigParser** → brancher les vrais ports, routes, `root`.
4. **POST (upload) + DELETE.**
5. **CGI.**
6. **Durcissement** : codes exacts, `client_max_body_size`, autoindex, redirections, `error_page`, keep-alive, timeouts, stress-test vs NGINX.

À chaque palier le serveur **marche**, juste avec moins de fonctionnalités. On ne code jamais trois semaines à l'aveugle.

---

## Le résumé en 5 phrases

1. Au démarrage on lit la config et on ouvre des sockets d'écoute.
2. Une boucle infinie tourne autour d'**un seul `poll()`** qui dit quels fd sont prêts.
3. Quand un client envoie des octets, on les donne au parseur **qui reprend où il en était** ; tant que la requête est incomplète, on attend le tour suivant.
4. Requête complète → routing (quel fichier ? quelle méthode ? autorisé ?) → traitement (lire le fichier, lancer le CGI…) → **`Response` emballe le résultat** en octets HTTP.
5. On met ces octets dans `out_buffer`, on demande à `poll` de prévenir quand on peut écrire, et on envoie par petits bouts jusqu'à la fin — puis keep-alive ou `close`.

---

*Voir aussi : `fiche_webserv_conceptuelle.md` (le détail de chaque mécanisme), `plan_repartition_binome.md` (qui fait quoi), et `matiere http/COMPRENDRE_LA_RESPONSE.md` (ta pièce en particulier).*
