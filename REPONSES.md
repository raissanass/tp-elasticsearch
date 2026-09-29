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



