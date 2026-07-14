# Règles de conception technique

Règles opposables en revue de code pour des services Java qui manipulent des
fichiers volumineux et des flux d'événements Kafka. Chaque règle a un
identifiant (R1, R2…) pour être citée dans les PR et les ADR. Les identifiants
sont repris dans `metadata.regle` et `metadata.reference` des règles Semgrep
de ce pack (`a-memoire-fichiers.yaml`, `b-kafka.yaml`), qui n'en couvrent que
le sous-ensemble détectable statiquement — R4, R9, R11 restent des règles de
conception non outillées ici.

---

## A. Gestion mémoire et fichiers

### R1 — Aucun fichier chargé entièrement en mémoire

Interdits sur tout flux de fichier volumineux (document, pièce jointe,
archive) :

- `byte[] content = inputStream.readAllBytes()` / `Files.readAllBytes(...)`
- `ByteArrayOutputStream` accumulant un fichier complet
- `String xml = new String(bytes)` sur un document entier
- Parsing DOM (`DocumentBuilder`) ou unmarshalling JAXB d'un document non borné

Obligatoire :

- Traitement par flux : `InputStream` → traitement → `OutputStream`, buffers
  bornés (8–64 Ko).
- XML : **StAX** (`XMLStreamReader`) ou SAX pour la validation et
  l'extraction. JAXB acceptable uniquement sur des sous-arbres bornés extraits
  via StAX (ex. un élément métier unitaire), jamais sur le document racine
  d'un flux entrant.
- Empreintes (SHA-256) calculées au fil de l'eau (`DigestInputStream`)
  pendant l'upload, pas dans une passe séparée en mémoire.

### R2 — Upload et download en streaming pur

- HTTP entrant : multipart/streaming consommé au fil de l'eau et **écrit
  directement, au fil de l'eau, vers sa destination finale**. Le corps de
  requête ne transite jamais par un buffer complet en heap.
- Taille max de requête imposée au niveau du serveur ET vérifiée au fil de
  l'eau (compteur d'octets) — ne pas faire confiance au `Content-Length`.
- Download/restitution : proxy en streaming source → client, sans
  matérialisation intermédiaire complète.

### R3 — Bornes explicites partout

Tout ce qui est non borné finit par faire tomber un pod. Imposer :

- taille max de fichier acceptée (et par type de contenu) ;
- profondeur et nombre d'éléments max en parsing XML (protection XXE et
  « billion laughs » : `XMLInputFactory` avec `SUPPORT_DTD=false` et
  `IS_SUPPORTING_EXTERNAL_ENTITIES=false`) ;
- taille max des collections chargées depuis la DB (pagination obligatoire,
  pas de `findAll()`) ;
- pools bornés (connexions DB, clients HTTP, threads).

Si une interface externe (partenaire, régulateur) impose ses propres limites
(taille de flux, taille par fichier, quotas), les documenter et les
reprendre comme point de départ pour dimensionner les bornes internes du
service — elles ne dispensent pas de bornes propres, potentiellement plus
strictes, sur les flux internes non contraints par cette interface.

### R4 — Dimensionnement JVM standardisé

- Java 21+, G1 par défaut ; ZGC pour les services sensibles à la latence.
- Heap plafonnée par service et cohérente avec les limites mémoire du
  conteneur : `-Xmx` explicite, ou à défaut `MaxRAMPercentage` ≤ 75 % de la
  limite du pod (les deux sont exclusifs — `-Xmx` prime s'il est posé).
- Les I/O bloquantes (DB, HTTP) tournent sur **virtual threads** — pas de
  réactif complexe pour compenser des lectures en mémoire qu'on n'aurait pas
  dû faire.
- Tout service traitant des fichiers doit avoir un test de charge prouvant
  une consommation heap **constante** quelle que soit la taille du fichier
  (fichier de 1 Ko vs 100 Mo → même profil mémoire).

---

## B. Kafka et flux événementiels

### R5 — Claim check : jamais de payload fichier dans Kafka

Un événement porte : identifiants métier, une référence vers l'emplacement où
le fichier est stocké (clé/identifiant — fichier brut, et le cas échéant jeu
de données extrait), `sha256`, taille, format, horodatage. Jamais le
contenu. Taille max de message applicative : 256 Ko (marge sous la limite
broker de 1 Mo).

### R6 — Partitionnement par clé métier, ordre garanti par entité

- Clé de partition : un identifiant métier stable (ex. l'identifiant de
  l'entité concernée, ou une clé de regroupement pour un topic de lot) pour
  garantir l'ordre des événements relatifs à une même entité.
- Ne jamais dépendre d'un ordre global inter-entités.
- Nombre de partitions dimensionné dès la création (ex. 48–96 sur les topics
  chauds) : le repartitionnement casse les garanties d'ordre.

### R7 — At-least-once + idempotence, pas de « exactly-once » magique

- Producteurs : `enable.idempotence=true`, `acks=all` (défauts depuis
  Kafka 3.0 — les poser explicitement en config pour qu'aucun réglage
  voisin ne les désactive silencieusement).
- Consommateurs : commit après traitement ; tout handler doit être
  **rejouable** — déduplication par clé métier (contrainte unique en DB ou
  table `processed_events`).
- Transactions Kafka réservées aux topologies Kafka→Kafka (Streams) ; pour
  DB+Kafka, c'est l'outbox (R8).

### R8 — Outbox pattern pour toute écriture DB + événement

Écriture de l'état et de l'événement dans la même transaction PostgreSQL
(table outbox), publication vers Kafka par un relais (Debezium ou poller).
La double écriture directe DB puis Kafka est interdite.

### R9 — Contrats d'événements versionnés

- Schema Registry + Avro (ou Protobuf), compatibilité `BACKWARD` imposée.
- Un topic = un contrat ; convention de nommage
  `<contexte>.<agrégat>.<fait>` (ex. `commande.paiement.effectue`).
- Les événements sont des **faits passés** immuables, nommés au participe
  passé — jamais des commandes déguisées.

### R10 — DLQ et rejeu systématiques

- Chaque consumer group a sa DLQ (`<topic>.<groupe>.dlq` — le groupe fait
  partie du nom : plusieurs groupes peuvent consommer le même topic) avec
  l'erreur et les en-têtes d'origine.
- Retries : d'abord in-process avec backoff (erreurs transitoires), puis DLQ.
  Jamais de retry infini qui bloque une partition.
- Procédure de rejeu documentée et testée (le rejeu est un cas nominal
  d'exploitation, pas un incident).

### R11 — Backpressure par le consommateur

Le débit d'entrée ne doit jamais imposer son rythme aux traitements aval :
Kafka est le tampon. Dimensionner par le **lag** (alerte + autoscaling des
consumers sur le lag), pas par des files en mémoire.
