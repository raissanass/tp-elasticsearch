### Exercice 1.1 — Explorer le cluster

**Quelle version tourne ?** Elasticsearch 9.5.4 (Lucene 10.5.1).

**Combien de nœuds ?** Un seul : `es01`. Il est à la fois maître (`*`) et porte tous les rôles (`cdfhilmrstw` : données, ingestion, master, etc.). C'est un cluster mono-nœud de labo.

**Pourquoi des index commençant par un point ?** Ce sont des index système ou cachés, créés automatiquement par Elasticsearch et Kibana pour leur fonctionnement interne : sécurité (`.security-7`), configuration et objets Kibana (`.kibana_*`), alertes (`.internal.alerts-*`), gestionnaire de tâches, journaux (`.ds-*`, qui sont des data streams)… Ils ne contiennent pas de données métier et sont masqués par défaut : il faut `expand_wildcards=all` pour les afficher. Tous sont `green` avec 1 shard primaire et 0 réplique, ce qui est cohérent avec un seul nœud.

### Exercice 1.2 — CRUD

**Comment évolue `_version` ?** Elle augmente de 1 à chaque écriture sur le document : création (1), mise à jour partielle (2), suppression (3). Même la suppression incrémente la version. Si on relance une requête d'écriture, la version augmente encore, même si le contenu est identique (dans mon cas, la suppression est arrivée en version 7 après plusieurs exécutions).

**Quel identifiant reçoit le document créé par `POST essai/_doc` ?** Un `_id` généré automatiquement par Elasticsearch : une chaîne aléatoire de 20 caractères (ici `1IRd7aABXNpuLRK6PP1c`). Avec `POST` sans id, on ne choisit pas l'identifiant.

**L'index `essai` existait-il avant le premier `PUT` ?** Non, il n'apparaissait pas dans `_cat/indices` à l'exercice 1.1. Elasticsearch l'a créé automatiquement à la première écriture, avec un mapping déduit (dynamique) et les réglages par défaut : 1 shard et 1 réplique. Comme il n'y a qu'un nœud, la réplique ne peut pas être placée : les écritures indiquent `"total": 2, "successful": 1`, et le cluster passe en `yellow`.

### Exercice 1.3 — Pièges du mapping dynamique

**Quel type reçoit `salaire` ?** `text`, avec un sous-champ `salaire.keyword` de type `keyword` (`ignore_above: 256`). La valeur `"45000"` étant envoyée entre guillemets, Elasticsearch la considère comme une chaîne. Il ne détecte pas les nombres dans les chaînes. Même chose pour `actif` : `"true"` devient `text` et non `boolean`.

**Et `publication` ?** `date` : le mapping dynamique détecte automatiquement les chaînes au format date (`2026-08-02`).

**Pourquoi le document 2 est-il accepté ?** Le type de `salaire` est déjà fixé à `text` par le premier document. Le nombre `52000` est alors converti en chaîne `"52000"` pour être indexé, donc il n'y a pas d'erreur.

**Conséquence pour un tri ou un filtre `salaire > 50000` ?** Les comparaisons se font sur des chaînes, donc dans l'ordre alphabétique et non numérique : `"100000"` est considéré comme inférieur à `"45000"`. Les tris et les filtres par intervalle donnent des résultats faux, et les agrégations numériques (moyenne, somme) sont impossibles. D'où l'intérêt d'un mapping explicite, car le type d'un champ ne peut plus être changé sans recréer l'index.


### Exercice 1.4 — Mapping explicite de l'index `offres`

**Choix des types :**
- `keyword` pour `id`, `entreprise`, `ville`, `contrat`, `teletravail` : valeurs exactes pour les filtres et les facettes.
- `titre` : `text` avec l'analyseur `french` pour la recherche, plus un sous-champ `brut` en `keyword` pour le tri et les facettes sur la valeur exacte.
- `description` : `text` avec l'analyseur `french`.
- `competences` : `keyword` pour les filtres et facettes, plus un sous-champ `texte` en `text` (analyseur `standard`, adapté aux noms techniques) pour la recherche plein texte.
- `localisation` : `geo_point` pour la recherche par distance.
- `experience_annees`, `salaire_min`, `salaire_max` : `integer` pour les intervalles, tris et moyennes.
- `date_publication` : `date` (format `yyyy-MM-dd`) pour les filtres par date et l'histogramme mensuel.
- Réglages : 1 shard, 0 réplique (un seul nœud, donc cluster `green`), `"dynamic": "strict"`.

**Quelle erreur obtenez-vous ?** Code **400**, `strict_dynamic_mapping_exception` : « [1:20] mapping set to strict, dynamic introduction of [champ_inconnu] within [_doc] is not allowed ». `[1:20]` indique la position dans le JSON envoyé (ligne 1, colonne 20). Le document entier est rejeté : `GET offres/_count` renvoie toujours 0.

**Pourquoi est-ce une bonne pratique en production ?**
- Une faute de frappe dans un nom de champ (`vile` au lieu de `ville`) est détectée immédiatement au lieu de créer silencieusement un nouveau champ.
- Aucun champ ne reçoit un type deviné à tort (cf. exercice 1.3), ce qui évite des tris et agrégations faux.
- Le mapping reste maîtrisé : pas d'explosion du nombre de champs, qui consommerait de la mémoire.
- Comme un type ne peut plus être changé sans recréer l'index, mieux vaut le décider soi-même dès le départ.

**Nettoyage :** `DELETE essai` et `DELETE essai2` → le cluster repasse en `green` (plus aucun shard non assigné).

## Partie 2 — Ingestion en Python

### Exercice 2.2 — Idempotence et identifiants

**Le nombre de documents a-t-il doublé ?** Non : après la deuxième exécution sans `--reset`, l'index contient toujours 5 000 documents. Le script annonce 5 000 documents indexés, mais chacun a remplacé le document existant de même `_id` (sa `_version` passe de 1 à 2, visible avec `GET offres/_doc/OFF-00002`).

**Pourquoi fixer `_id` à partir du champ `id` est-il essentiel ?** Avec l'action `index`, Elasticsearch remplace un document qui a le même `_id`. En utilisant l'identifiant métier (`OFF-00001`…), une même offre a toujours le même `_id` : relancer l'ingestion (après une panne, une mise à jour des données…) ne crée pas de doublons. Le script est idempotent.

**Que se passerait-il avec des identifiants générés par Elasticsearch ?** Chaque exécution créerait de nouveaux `_id` aléatoires : les 5 000 offres seraient ajoutées une deuxième fois, soit 10 000 documents, puis 15 000 à la troisième exécution, etc. On aurait des doublons impossibles à distinguer, qui fausseraient les recherches et les statistiques.

### Exercice 2.3 — Provoquer une erreur de mapping

**Le lot entier est-il rejeté ou seulement ce document ?** Seulement ce document. Le script affiche « 5000 documents indexés, 1 erreurs » : l'offre `OFF-99999` est refusée avec un statut 400 (`strict_dynamic_mapping_exception` : « dynamic introduction of [prime] within [_doc] is not allowed »), car le champ `prime` n'existe pas dans le mapping strict. Les autres documents du même lot sont bien indexés : l'API `_bulk` renvoie un statut par opération, et une erreur n'annule pas les autres.

**Intérêt de `raise_on_error=False` pour un pipeline :** avec la valeur par défaut (`True`), `helpers.bulk` lèverait une exception à la première erreur et le script s'arrêterait sans afficher le bilan. Avec `False`, l'ingestion va jusqu'au bout, et on récupère la liste des documents rejetés avec la raison du rejet. On peut alors les journaliser, les corriger ou les renvoyer plus tard, sans bloquer le chargement de milliers de documents valides à cause d'un seul document invalide.

### Exercice 2.4 — Vérifier dans Kibana

`GET _cat/indices/offres?v` : index `green`, 5 000 documents. `GET offres/_count` renvoie 5 000 et `GET offres/_doc/OFF-00002` renvoie l'offre « Analyste Cybersécurité Senior » (Cévennes Data).

Data view `offres` créée avec `date_publication` comme champ temporel. Avec la période « Last 1 year », Discover affiche 4 968 documents et non 5 000 : la période se termine à l'instant présent (29/09/2026), alors que 32 offres sont datées du 30/09/2026, donc exclues. En prolongeant la date de fin, on retrouve les 5 000 offres. Les dates, stockées en UTC à minuit, sont affichées à 02:00 car Kibana utilise le fuseau du navigateur (Paris, UTC+2).

## Partie 3 — Recherche et analyseurs

### Exercice 3.1 — Voir travailler un analyseur

**Tokens obtenus :**
- `standard` : `les`, `développeuses`, `travaillaient`, `sur`, `l'analyse`, `des`, `données` (découpage et minuscules seulement).
- `french` : `developeu`, `travailaient`, `analys`, `done`.

**Quels mots disparaissent avec `french` ?** Les mots vides : `les`, `sur`, `des`. Les positions des tokens restants sont conservées (1, 2, 4, 6), ce qui permet encore les recherches de phrase.

**Que devient `l'analyse` ?** Avec `standard`, c'est un seul token `l'analyse`. Avec `french`, l'élision retire `l'`, puis la racinisation donne `analys`.

**« donnée » et « données » donnent-ils le même terme ?** Avec `standard` : non (`donnée` ≠ `données`). Avec `french` : oui, les deux deviennent `done` (accents retirés, pluriel et féminin supprimés, lettres doublées réduites).

**Conséquence pour la recherche :** comme le même analyseur est appliqué au document et à la question, l'analyseur `french` permet de retrouver un document quelle que soit la forme du mot (singulier ou pluriel, masculin ou féminin, avec ou sans article élidé) : on gagne en rappel. C'est pourquoi `titre` et `description` utilisent `french`. En contrepartie, la racinisation peut rapprocher des mots différents et abîmer des termes techniques ; c'est pour cela que `competences.texte` utilise `standard`.

### Exercice 3.2 — `match` contre `term`

**Pourquoi les deux requêtes `term` renvoient-elles 0 résultat ?** `term` cherche la valeur exacte, sans aucune analyse.
- `ville` est un `keyword` : la valeur stockée est `"Paris"` avec une majuscule, donc `"paris"` ne correspond à rien.
- `titre` est un champ `text` analysé par `french` : l'index ne contient que des tokens (`data`, `engine`…), jamais la chaîne entière « Data Engineer Senior ». `term` ne doit pas être utilisé sur un champ `text`.

**Corrections :**
- `{ "term": { "ville": "Paris" } }` → 1 492 résultats.
- `{ "term": { "titre.brut": "Data Engineer Senior" } }` (sous-champ `keyword`) → 103 résultats.

**Effet de `"operator": "and"` :** la requête `match` sur « projets bancaires » passe de 4 190 à 393 résultats. Par défaut, `match` combine les tokens avec un OU (un seul mot suffit, et « projet » est très fréquent). Avec `and`, tous les tokens doivent être présents : on gagne en précision et on perd en rappel.


### Exercice 3.3 — Plusieurs champs, pondération et fautes de frappe

**Sans correction :** `multi_match` sur « kubernetis terraform » (champs `titre`, `competences.texte`, `description`) → 739 résultats. Le token `kubernetis` n'existe dans aucun document : seules les offres mentionnant Terraform (dans les compétences ou la description) sont trouvées. Beaucoup de scores sont identiques, car le type par défaut `best_fields` ne garde que le score du meilleur champ.

**Quel paramètre rattrape la faute ?** `"fuzziness": "AUTO"` : il tolère des différences de lettres (distance d'édition) selon la longueur du mot (0 erreur jusqu'à 2 lettres, 1 erreur de 3 à 5 lettres, 2 au-delà). `kubernetis` est à 1 lettre de `kubernetes` : on passe à 969 résultats, et le score maximal monte de 3,10 à 5,74, car les deux mots contribuent désormais.

**Effet du poids `titre^3` :** sur « kubernetis terraform », aucun changement (mêmes résultats, mêmes scores), car aucun titre ne contient ces mots : multiplier un score nul ne change rien. Sur « devops kubernetis », le poids change nettement l'ordre : sans poids, les « Architecte Cloud » et « Ingénieur DevOps » sont mélangés en tête ; avec `titre^3`, les offres dont le titre contient « DevOps » passent toutes devant. Le poids sert à dire qu'un mot trouvé dans le titre est plus significatif que le même mot trouvé dans une longue description.

### Exercice 3.4 — Requête `bool`

Requête : `must` = `multi_match` « données » sur `titre` et `description` ; `filter` = `term` contrat CDI, `terms` ville [Montpellier, Toulouse], `range` salaire_max ≥ 50 000 ; `must_not` = `term` teletravail « aucun » ; `should` = `term` competences « Elasticsearch ».

**Comparaison des `_score` avec et sans `should` :** 25 résultats dans les deux cas : le `should` n'est pas obligatoire quand un `must` ou un `filter` est présent, il n'exclut rien. En revanche, il modifie le classement : les meilleures offres passent de 2,05 à 4,01, soit un bonus d'environ 1,97 pour les offres qui demandent Elasticsearch. Elles remontent donc en tête.

**Pourquoi placer les critères exacts dans `filter` plutôt que dans `must` ?**
1. **Pertinence :** en contexte filtre, aucun score n'est calculé. Le classement ne dépend alors que de la pertinence textuelle (« données »), sans être faussé par des critères oui/non comme le contrat ou la ville.
2. **Performance :** ne pas calculer de score est plus rapide, et les résultats d'un filtre peuvent être mis en cache par Elasticsearch, puis réutilisés par les requêtes suivantes qui ont le même filtre.

### Exercice 3.5 — Recherche géographique

Requête : `geo_distance` de 20 km autour de (43.6108, 3.8767) dans un `filter`, et tri `_geo_distance` croissant en km.

**Résultats :** 340 offres à moins de 20 km de Montpellier. La plus proche est à environ 0,19 km du point. Le tableau `sort` de chaque résultat contient la distance calculée, dans l'unité demandée (`unit: "km"`).

**Le `_score` vaut `null` :** la requête n'utilise qu'un filtre (pas de calcul de pertinence) et le tri est imposé par la distance, pas par le score. Elasticsearch ne calcule donc pas le score, qui ne servirait pas à classer les résultats.

### Exercice 3.6 — Pagination et surlignage

Requête 3.4 reprise avec `"from": 5, "size": 5` (page 2, 5 résultats par page : `from` = (page − 1) × size), `_source` limité à `titre`, `entreprise`, `ville`, et un `highlight`.

**Résultats :** `hits.total.value` reste à 25 (la pagination ne change pas le total), mais seuls 5 documents sont renvoyés. Avec un surlignage sur `description` seulement, aucun bloc `highlight` n'apparaissait : ces offres correspondent grâce à leur titre (« Administrateur Bases de Données »), pas à leur description. En ajoutant `titre` au surlignage, on obtient `"Administrateur Bases de <em>Données</em> Lead"` : les termes trouvés sont entourés de balises `<em>`, prêtes à être affichées dans une page web.

**Pourquoi `from` + `size` est-il limité à 10 000 par défaut ?** Pour afficher les résultats à partir de la position `from`, chaque shard doit trouver et trier ses `from + size` meilleurs documents, puis le nœud coordinateur doit fusionner et trier tous ces résultats avant d'en jeter la plus grande partie. Plus on va loin, plus cela coûte en mémoire et en calcul : une pagination profonde pourrait surcharger le cluster. Le réglage `index.max_result_window` fixe donc la limite à 10 000.

**Quelle API utiliser au-delà ?** `search_after` avec un point in time (PIT) : on ouvre un PIT (`POST offres/_pit?keep_alive=1m`), qui fige une vue cohérente de l'index, puis on demande la page suivante en passant les valeurs de tri du dernier résultat de la page précédente (`search_after`). Il n'y a plus de documents à sauter : chaque page coûte le même prix, quelle que soit sa profondeur.

## Partie 4 — Agrégations

### Exercice 4.1 — Offres et salaire moyen par ville

**Quelle ville a le salaire moyen le plus élevé ?** Paris, avec un `salaire_min` moyen d'environ 57 442 €, devant Grenoble (53 046 €) et Nantes (52 125 €).

**Sur combien d'offres la moyenne est-elle calculée ?** Pas sur toutes : à Paris, 994 offres sur 1 492 ont un `salaire_min` (mesuré avec une sous-agrégation `value_count`). Le champ est absent pour certains contrats, et `avg` ignore les documents qui n'ont pas le champ, sans les compter comme des zéros. Le `doc_count` d'un paquet et le nombre de valeurs utilisées pour la moyenne sont donc différents.

**Erreur en remplaçant `ville` par `titre` :** `illegal_argument_exception` : « Fielddata is disabled on [titre] ». `titre` est un champ `text` : il est stocké en tokens dans un index inversé, adapté à la recherche mais pas au regroupement ni au tri. Activer `fielddata` est déconseillé (forte consommation mémoire, et regroupement par mots isolés au lieu des titres entiers). La correction consiste à agréger sur le sous-champ `keyword` : `"field": "titre.brut"`.

### Exercice 4.2 — Publications par mois

Requête : `date_histogram` sur `date_publication` avec `"calendar_interval": "month"` (et `"format": "yyyy-MM"` pour des clés lisibles), contenant une sous-agrégation `terms` sur `contrat`.

**Résultats :** 6 mois, d'avril à septembre 2026, pour un total de 5 000 offres : 763 (avril), 865 (mai), 820 (juin), 835 (juillet), 880 (août, le maximum), 837 (septembre). Avril est un peu plus bas car la période couverte commence début avril.

**Ventilation par contrat :** chaque mois, le CDI est largement majoritaire (environ 55 %, par exemple 476 sur 880 en août), suivi de l'alternance, puis du CDD et du freelance à des niveaux proches ; le stage est le contrat le plus rare. La répartition reste stable d'un mois à l'autre.

**Remarque :** `calendar_interval` respecte les mois réels (28 à 31 jours), contrairement à `fixed_interval` qui découpe en durées fixes (par exemple 30 jours).

### Exercice 4.3 — Tranches de salaire et statistiques

Requête : agrégation `range` sur `salaire_min` (`from` inclus, `to` exclu) avec trois tranches nommées, et agrégation `stats` sur `experience_annees`.

**Tranches de salaire :** < 40 k : 484 offres ; 40–55 k : 1 363 ; ≥ 55 k : 1 542. Le total (3 389) est inférieur à 5 000 : les 1 611 offres sans `salaire_min` n'entrent dans aucune tranche. Parmi les offres qui affichent un salaire, la tranche ≥ 55 k est la plus représentée (environ 45 %).

**Statistiques d'expérience :** calculées sur les 5 000 offres (le champ est toujours présent) : minimum 0 an, maximum 15 ans, moyenne d'environ 5,9 ans (somme 29 564). L'agrégation `stats` fournit ces cinq valeurs en une seule fois.

### Exercice 4.4 — Requête + agrégation

Requête : `match_phrase` sur `titre` « Data Engineer » (les deux mots doivent se suivre, contrairement à `match` qui accepterait « Data Analyst »), avec `"size": 0`, une agrégation `terms` sur `competences` (`size: 5`) et une agrégation `terms` sur `teletravail`.

**Résultats (462 offres) :** les 5 compétences les plus demandées sont Airflow (315), Spark (313), Kafka (312), Python (311) et SQL (301). Le télétravail le plus fréquent est `partiel` (284 offres, soit 61 %), devant `aucun` (109) et `total` (69).

**L'agrégation porte-t-elle sur tout l'index ou seulement sur les résultats de la requête ?** Seulement sur les résultats de la requête. La même agrégation sans `query` (5 000 offres) donne un top 5 totalement différent : Python (1 320), Elasticsearch (1 279), Linux (1 003), Git (1 000), Docker (986). Airflow, Spark et Kafka, caractéristiques du métier de Data Engineer, n'y figurent pas. La répartition du télétravail est en revanche proche (61 % de `partiel` dans les deux cas).

## TP2
## Exercice 0 — Premier pipeline

**Quels champs Logstash a-t-il ajoutés à la phrase ?** La phrase tapée est placée dans `message`. Logstash ajoute :
- `@timestamp` : horodatage de l'événement ;
- `@version` : version du format d'événement (`"1"`) ;
- `event.original` : copie brute de la ligne lue (nommage ECS) ;
- `host.hostname` : nom de la machine qui a produit l'événement, ici l'identifiant du conteneur Docker.

Avec le filtre `mutate { uppercase => ["message"] }`, `message` passe en majuscules, mais `event.original` reste inchangé : il conserve la donnée d'origine.

**Que contient `@timestamp` ?** L'heure à laquelle Logstash a **reçu** l'événement (quand on a appuyé sur Entrée), en UTC (suffixe `Z`) : par exemple `08:44:15Z`, soit 10:44 à Paris. Ce n'est pas l'heure à laquelle l'événement s'est réellement produit. Pour des logs, il faudra la remplacer par la date écrite dans la ligne (filtre `date`, partie 3).

**À quoi sert `--path.data /tmp/essai` ?** Logstash utilise un dossier de données (file d'attente, sincedb, verrou) et refuse de démarrer si une autre instance utilise déjà le même dossier. Le service `logstash` de la stack utilise `/usr/share/logstash/data` (volume `lsdata`) : en donnant un dossier séparé et jetable à ce Logstash d'essai, on évite le conflit de verrou et on ne mélange pas son état avec celui du vrai service.

## Partie 1 — Recharger les offres avec Logstash

### Exercice 1.2 — Premier lancement

**Les documents sont-ils indexés ?** Non. Chaque offre est refusée : Logstash journalise « Could not index event to Elasticsearch » avec un statut **400** et une `strict_dynamic_mapping_exception` (« mapping set to strict, dynamic introduction of [host] within [_doc] is not allowed »). Le `_version` de `OFF-00002` ne change pas.

**Quels noms de champs sont en cause ?** Elasticsearch cite `host`, le premier champ inconnu rencontré. L'événement contient en réalité cinq champs ajoutés par Logstash qui n'existent pas dans le mapping : `@timestamp`, `@version`, `host` (`host.name`), `log` (`log.file.path`) et `event` (`event.original`).

**Lien avec `"dynamic": "strict"` :** comme à l'exercice 1.4 du TP d'introduction, le mapping strict rejette tout document qui contient un champ non déclaré. Les données de l'offre sont correctes, mais les métadonnées ajoutées par Logstash (nommage ECS) suffisent à faire refuser tout le document.

### Exercice 1.3 — Corriger

Ajout dans `filter` de `mutate { remove_field => ["@timestamp", "@version", "event", "log", "host"] }` : on supprime les champs ajoutés par Logstash que le mapping strict refuse.

**Le nombre de documents a-t-il changé ?** Non : toujours 5 000.

**Et le `_version` de `OFF-00002` ?** Il passe de 3 à 4. Grâce à `document_id => "%{id}"`, chaque offre est envoyée avec son identifiant métier : Elasticsearch remplace le document existant de même `_id` au lieu d'en créer un nouveau. Le `_version` augmente, mais le nombre de documents reste le même.

**Pourquoi supprimer ces champs plutôt qu'assouplir le mapping ?** Ces champs sont des métadonnées techniques de Logstash (heure de lecture, nom du conteneur, chemin du fichier, ligne brute) qui n'ont aucun sens métier pour une offre d'emploi. Assouplir le mapping (`dynamic: true`) les ajouterait à chaque offre, alourdirait l'index (`event.original` duplique toute la ligne) et ferait perdre la protection du mode strict contre les champs inattendus (fautes de frappe, données mal formées). Le mapping doit décrire les données métier ; c'est au pipeline de s'y adapter.

**Pourquoi l'index `offres` doit-il exister avant le premier démarrage de Logstash ?** Avec `manage_template => false`, Logstash n'installe aucun modèle d'index. Si l'index n'existait pas, la première écriture le créerait automatiquement avec un mapping dynamique : `ville` ou `contrat` deviendraient `text` + `keyword`, `localisation` un objet de deux nombres au lieu d'un `geo_point`, l'analyseur `french` serait absent, et tous les champs ajoutés par Logstash seraient acceptés. Les recherches géographiques et les agrégations du TP d'introduction ne fonctionneraient plus correctement.

### Exercice 1.4 — Relancer

Après un nouveau redémarrage : toujours 5 000 documents, et le `_version` de `OFF-00002` passe à 5.

**Combien de fois le fichier a-t-il été lu ?** Une fois par démarrage de Logstash : avec `sincedb_path => "/dev/null"`, Logstash ne mémorise pas sa position de lecture, donc il relit `offres.ndjson` entièrement à chaque démarrage. Le fichier a été lu trois fois (exercices 1.2, 1.3 et 1.4) ; le `_version` augmente de 1 à chaque lecture réussie.

**Avec la sincedb par défaut ?** Logstash enregistrerait dans un fichier sincedb (dans son dossier de données, le volume `lsdata`) qu'il a déjà lu `offres.ndjson` jusqu'au bout. Au redémarrage suivant, il ne relirait pas le fichier : `_version` resterait à 4. C'est le comportement souhaité en production, où l'on ne veut traiter chaque fichier qu'une fois.

**Et si `document_id` n'était pas renseigné ?** Elasticsearch générerait un `_id` aléatoire pour chaque document : chaque lecture du fichier ajouterait 5 000 nouvelles offres (10 000 au 2e démarrage, 15 000 au 3e…). L'ingestion ne serait plus idempotente, et les recherches et statistiques seraient faussées par les doublons.

## Partie 2 — Superviser et fiabiliser
### Exercice 2.1 — Superviser

**Pipelines chargés :** deux, `offres` et `web`, déclarés dans `pipelines.yml`. Chacun a 4 workers (par défaut, le nombre de cœurs du processeur), des lots de 125 événements et un délai d'attente de 50 ms. La DLQ est désactivée pour les deux.

**Compteurs du pipeline `offres` :** `in` = 5 000 (événements lus par l'entrée), `filtered` = 5 000 (passés par les filtres), `out` = 5 000 (envoyés à la sortie). Ces compteurs sont remis à zéro à chaque démarrage de Logstash : ils correspondent à une seule lecture complète du fichier, celle du dernier redémarrage. La sortie a envoyé 68 requêtes `_bulk`, toutes acceptées (code 200).

**Plugin le plus coûteux :** la sortie `elasticsearch`, avec 142 829 ms cumulés, contre 11 284 ms pour le filtre `mutate`. L'envoi par le réseau et l'attente de l'indexation par Elasticsearch coûtent bien plus cher que la suppression de quelques champs en mémoire. Ces durées sont cumulées sur tous les workers : ce n'est pas le temps réel écoulé.

### Exercice 2.2 — Isoler les documents rejetés

Mise en œuvre : ajout de `DEAD_LETTER_QUEUE_ENABLE=true` dans l'environnement du service `logstash` (`docker-compose.override.yml`), chemin d'entrée élargi à `/data/offres*.ndjson` dans `offres.conf`, puis création de `data/offres_test.ndjson` contenant l'offre `OFF-99999` avec un champ `"prime": 3000`.

**Le document `OFF-99999` est-il dans l'index ?** Non : `GET offres/_doc/OFF-99999` renvoie `"found": false`, et `_count` reste à 5 000. L'API de supervision indique pourtant `in` = 5 001 : le document a bien été lu et traité.

**Où se trouve-t-il ?** Dans la DLQ du pipeline `offres`, sur disque : `/usr/share/logstash/data/dead_letter_queue/offres/1.log` (1 618 octets ; volume `lsdata`). L'API de supervision indique une DLQ de 1 619 octets et `dropped_events: 0`. Il n'y a plus de message « Could not index » dans le journal : le document est rangé dans la DLQ au lieu d'être signalé puis perdu.

**Raison de refus enregistrée dans `[@metadata][dead_letter_queue]` :** `reason` = « Could not index event to Elasticsearch. status: 400 … strict_dynamic_mapping_exception … mapping set to strict, dynamic introduction of [prime] within [_doc] is not allowed », avec `plugin_type: elasticsearch`, l'identifiant du plugin et l'heure du refus (`entry_time`). `[@metadata][path]` indique le fichier source.

**Qu'apporte la DLQ par rapport à `raise_on_error=False` dans `ingest.py` ?** `raise_on_error=False` évite l'arrêt du script et affiche les erreurs, mais le document refusé n'est conservé nulle part : une fois le script terminé, il est perdu, il faut relire la sortie et retrouver la ligne à la main. La DLQ conserve durablement sur disque le document complet, avec la raison, la date et le plugin concerné ; elle survit aux redémarrages et peut être relue automatiquement par un pipeline Logstash (entrée `dead_letter_queue`) pour analyser, corriger et réinjecter les documents.

**Corriger et réinjecter en trois étapes :**
1. **Analyser** : relire la DLQ (entrée `dead_letter_queue`, `pipeline_id => "offres"`) et identifier la cause dans `[@metadata][dead_letter_queue][reason]` : ici, le champ `prime` absent du mapping.
2. **Corriger** : décider du traitement. Soit retirer le champ dans un pipeline de reprise (`mutate { remove_field => ["prime", "@timestamp", "@version"] }`), soit, si la prime est une donnée métier utile, l'ajouter d'abord au mapping (`PUT offres/_mapping` avec `"prime": { "type": "integer" }`).
3. **Réinjecter** : envoyer les événements corrigés avec la même sortie `elasticsearch` (index `offres`, `document_id => "%{id}"`), avec `commit_offsets => true` pour ne pas les retraiter. Vérifier ensuite avec `GET offres/_doc/OFF-99999` et `GET offres/_count` (5 001).

### Exercice 2.3 — Pourquoi deux pipelines ?

**Sans `pipelines.yml`, combien de pipelines ?** Un seul, nommé `main` : l'image Docker charge tous les fichiers du dossier `pipeline/` (`offres.conf` et `web.conf`) et les concatène en une seule configuration.

**Que deviendrait une offre lue dans `offres.ndjson` ?** Dans un pipeline unique, chaque événement passe par tous les filtres et part vers toutes les sorties. L'offre serait envoyée à la fois dans l'index `offres` et dans le data stream `logs-web-default`. Elle subirait aussi les filtres de `web.conf` (`grok`, `date`…), conçus pour des lignes de log : échec de `grok` (étiquette `_grokparsefailure`) et données polluées.

**Et une ligne de log d'accès ?** Même chose dans l'autre sens : elle serait envoyée dans `logs-web-default`, mais aussi dans l'index `offres`, où elle serait refusée par le mapping strict (champs inconnus) ou, sans `id`, indexée avec un `_id` littéral `%{id}` qui l'écraserait à chaque ligne.

**Deux autres avantages à isoler les pipelines :**
- **Isolation des pannes** : si Elasticsearch refuse ou ralentit les écritures d'un flux, ou si un pipeline plante, l'autre continue de fonctionner ; chacun a sa propre file d'attente.
- **Réglages et supervision séparés** : nombre de workers, taille de lots, type de file (mémoire ou persistée) et DLQ peuvent être ajustés par flux ; l'API de supervision donne des compteurs distincts (`in`, `out`, durées) pour chaque pipeline, ce qui facilite le diagnostic.
- (Plus de lisibilité aussi : chaque fichier ne contient que la logique de son flux, sans conditions `if` pour aiguiller les événements.)

### Exercice 2.4 — Ne rien perdre (réflexion)

**Avec la file en mémoire, que deviennent les événements lus mais pas encore envoyés lors d'un `docker kill` ?** Ils sont perdus : la file d'attente entre l'entrée et les filtres est en mémoire vive, et un arrêt brutal ne laisse pas à Logstash le temps de vider cette file vers Elasticsearch.

**Quel réglage change ce comportement ?** `queue.type: persisted` (dans `logstash.yml` ou `pipelines.yml`, par pipeline). La file est alors écrite sur disque : les événements reçus et non encore envoyés survivent à l'arrêt et sont traités au redémarrage. On obtient une garantie de livraison **« au moins une fois »** (*at-least-once*) : aucun événement n'est perdu, mais un événement peut être envoyé deux fois (par exemple s'il a été indexé juste avant l'arrêt, sans que Logstash ait eu le temps de le marquer comme traité).

**Pourquoi le `document_id` devient-il alors indispensable ?** Puisqu'un même événement peut être envoyé plusieurs fois, il faut que ces renvois ne créent pas de doublons. Avec `document_id => "%{id}"`, un renvoi remplace simplement le document de même `_id` : l'ingestion est idempotente, et la garantie « au moins une fois » devient en pratique « exactement une fois » dans l'index.

## Partie 3 — Transformer les logs d'accès
### Exercice 3.1 — Générer les logs

`python data/generate_access_logs.py` → `data/access.log`, 20 700 lignes au format Apache combined (23/09/2026 au 29/09/2026). Fichier exclu de Git (il se régénère).

### Exercice 3.2 — Mettre au point le motif

**Champs extraits par `%{COMBINEDAPACHELOG}` (Grok Debugger) :** `clientip` (203.0.113.123), `ident` et `auth` (`-`), `timestamp` (23/Sep/2026:00:00:39 +0200), `verb` (GET), `request` (/offres/OFF-01468), `httpversion` (1.1), `response` (200), `bytes` (43686), `referrer` et `agent`. Le Grok Debugger utilise les noms historiques ; dans Logstash 9 (mode ECS), le même motif produit `source.address`, `http.request.method`, `url.original`, `http.version`, `http.response.status_code`, `http.response.body.bytes`, `http.request.referrer` et `user_agent.original`.

**Type de `http.response.status_code` :** dans le Grok Debugger, `response` est une chaîne (`"200"`) : grok extrait du texte par défaut. La version ECS du motif utilisée par Logstash convertit le code et la taille en entiers (vérifié à l'exercice 3.4 : type `long`).

**Pourquoi `timestamp` doit-il encore être traité ?** C'est une simple chaîne. Sans le filtre `date`, `@timestamp` contiendrait l'heure de lecture par Logstash : toutes les requêtes sembleraient avoir eu lieu au même moment. Le filtre `date` lit la chaîne avec le format `dd/MMM/yyyy:HH:mm:ss Z` (mois en anglais, d'où `locale => "en"`) et la place dans `@timestamp`.

**Motif pour extraire `OFF-01468` de `/offres/OFF-01468/postuler` :** motif personnalisé `OFFRE_ID OFF-[0-9]{5}` et motif `^/offres/%{OFFRE_ID:offre_id}` → `{ "offre_id": "OFF-01468" }`. Dans `web.conf`, le champ cible est `[labels][offre_id]`.

### Exercice 3.3 — `web.conf`

- `grok` avec `%{COMBINEDAPACHELOG}` : découpe la ligne en champs ECS.
- `date` sur `timestamp` (format `dd/MMM/yyyy:HH:mm:ss Z`, `locale => "en"`) : date réelle de la requête dans `@timestamp`, puis suppression de `timestamp`.
- `useragent` sur `[user_agent][original]` : navigateur, système et appareil.
- Condition `if [url][original] =~ /^\/offres\/OFF-/` : second `grok` avec le motif `OFFRE_ID` → `[labels][offre_id]` (8 511 événements concernés).
- Suppression de `[event][original]` seulement si `grok` a réussi (on garde la ligne brute en cas d'échec pour diagnostiquer).
- Sortie `elasticsearch` en data stream : `data_stream_type => "logs"`, `data_stream_dataset => "web"`, `data_stream_namespace => "default"` → `logs-web-default`.

L'API de supervision confirme 20 700 événements en entrée et en sortie du pipeline `web`.

### Exercice 3.4 — Vérifier le data stream

**Documents et échecs de grok :** 20 700 documents et 0 événement portant l'étiquette `_grokparsefailure`. (Un premier essai a donné 20 841 documents : Logstash tournait pendant la génération de `access.log`, et 141 lignes avaient été lues une première fois. Après `DELETE _data_stream/logs-web-default` et un redémarrage propre : 20 700.)

**Index caché :** `.ds-logs-web-default-2026.10.01-000001`. `.ds-` est le préfixe des index de data stream (le point les rend cachés), `logs-web-default` le nom du data stream (`type-dataset-namespace`), `2026.10.01` la date de création de l'index (et non celle des logs), et `000001` le numéro de génération, incrémenté à chaque rollover géré par la politique `logs`.

**Premier événement :** `@timestamp` = `2026-09-22T22:00:39.000Z`, soit le 23/09/2026 à 00:00:39 (+02:00) : le filtre `date` a bien fonctionné. L'événement contient `source.address`, `http.request.method`, `url.original`, `user_agent.name` = Chrome Mobile, `user_agent.os.name` = Android et `labels.offre_id` = `OFF-01468`.

**Type de `http.response.status_code` :** `long`. C'est un nombre : on peut filtrer par intervalle (`>= 500`), faire des tranches et des statistiques ; en texte, les comparaisons seraient alphabétiques.

**`index.mode` :** `logsdb`, le mode de stockage compact dédié aux logs, appliqué par défaut aux data streams `logs-*-*` depuis Elasticsearch 9.0. (Le data stream est `YELLOW` car le modèle `logs` prévoit une réplique, impossible à placer sur un seul nœud.)

### Exercice 3.5 — Rejouer sans doublon ?

**Constat :** après un redémarrage de Logstash, le nombre de documents passe de 20 700 à 41 400 : le fichier a été relu entièrement (`sincedb_path => "/dev/null"`) et chaque ligne a été ajoutée une seconde fois.

**Pourquoi le problème ne se posait-il pas pour `offres` ?** La sortie du pipeline `offres` fixe `document_id => "%{id}"` : une relecture remplace le document de même `_id`. La sortie du pipeline `web` ne fixe pas de `_id` : Elasticsearch en génère un nouveau, aléatoire, à chaque envoi.

**Peut-on mettre à jour ou remplacer un document dans un data stream ?** Non, pas par l'écriture normale : un data stream est en ajout seul. Il n'accepte que l'opération `create` ; on ne peut pas remplacer un document existant. Il faut passer par `_update_by_query` / `_delete_by_query` ou viser l'index caché. C'est adapté aux logs, écrits une fois et jamais modifiés.

**Deux solutions pour rejouer sans doublon :**
1. **La sincedb** : avec une sincedb persistante (chemin par défaut, dans le volume `lsdata`, au lieu de `/dev/null`), Logstash mémorise qu'il a lu `access.log` jusqu'au bout et ne le relit pas au redémarrage.
2. **Un `_id` calculé avec `fingerprint`** : le filtre calcule une empreinte SHA-256 de la ligne (`message`), placée dans `[@metadata][fingerprint]`, puis la sortie utilise `document_id => "%{[@metadata][fingerprint]}"`. Une même ligne donne toujours le même `_id` : un renvoi est refusé comme doublon (conflit 409 sur `create`) au lieu d'être ajouté.

Remise en état : `docker compose stop logstash`, `DELETE _data_stream/logs-web-default`, `docker compose up -d logstash` → 20 700 documents.
