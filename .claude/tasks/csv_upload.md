# Plan de développement : Ingestion CSV dans le pipeline RAGpy

## Objectif
Permettre l'ajout direct de fichiers CSV dans le pipeline RAGpy, en contournant l'étape OCR et en reprenant le flux à partir de la variable `texteocr`. Le pipeline doit être refactorisé pour dépendre uniquement de `texteocr` et accepter des métadonnées arbitraires.

## Principe directeur
Toutes les sources de données (PDF/OCR, CSV, futures sources) convergent vers :
- Une variable unique : `texteocr` (contenu textuel)
- Un dictionnaire générique : `meta` (métadonnées arbitraires)

---

## Phase 1 : Cadrage et cartographie (Jour 1)

### 1.1 Analyse du pipeline actuel
**Objectif** : Documenter l'architecture existante

**Actions** :
- [ ] Cartographier `scripts/rad_chunk.py`
  - Identifier le point d'entrée principal
  - Tracer le flux de données de bout en bout
  - Documenter les dépendances entre fonctions

- [ ] Mapper les modules OCR
  - Localiser où l'OCR est appelé (Mistral Vision, autres services)
  - Identifier les fonctions qui produisent `texteocr`
  - Noter les transformations appliquées au texte brut

- [ ] Analyser l'injection de `texteocr`
  - Trouver tous les points où `texteocr` est créé
  - Identifier tous les points où `texteocr` est consommé
  - Vérifier les dépendances implicites (ex: suppositions sur PDF)

- [ ] Étudier la construction des métadonnées
  - Lister tous les champs `meta` actuellement utilisés
  - Identifier où et comment `meta` est construit
  - Vérifier s'il existe des validations/contraintes sur `meta`

- [ ] Examiner le stockage vectoriel
  - Identifier le connecteur (Pinecone, autre)
  - Comprendre le schéma d'indexation
  - Vérifier si des champs de `meta` sont whitlistés/requis

**Livrables** :
- Document markdown : `.claude/task/pipeline_current_architecture.md`
  - Diagramme de flux textuel
  - Tableau des fonctions clés avec leurs responsabilités
  - Liste des fichiers critiques avec annotations

---

## Phase 2 : Design d'ingestion CSV (Jour 1-2)

### 2.1 Spécification du point d'entrée CSV

**Objectif** : Définir l'interface d'ingestion CSV

**Actions** :
- [ ] Créer le module `ingestion/csv_ingestion.py`
  - Fonction principale : `ingest_csv(csv_path: str | pd.DataFrame, config: dict) -> List[dict]`
  - Retour : Liste de documents avec structure `{"texteocr": str, "meta": dict}`

- [ ] Définir le schéma minimal requis
  - Colonne obligatoire : `text` ou `content` (configurable)
  - Toutes autres colonnes → métadonnées automatiques
  - Support des alias via config (ex: `text_column="description"`)

- [ ] Implémenter les validations
  - Vérifier l'existence du fichier CSV
  - Détecter l'encodage automatiquement (chardet) avec fallback UTF-8
  - Valider la présence de la colonne texte
  - Alerter si CSV vide ou sans colonnes
  - Logger les statistiques (nombre de lignes, colonnes détectées)

### 2.2 Configuration du mode d'ingestion

**Objectif** : Permettre le choix de la source de données

**Actions** :
- [ ] Ajouter paramètre `source_type` dans la config
  - Valeurs acceptées : `"pdf"`, `"csv"`, `"auto"` (détection par extension)
  - Variable d'environnement : `RAGPY_SOURCE_TYPE`

- [ ] Créer un fichier de config dédié : `config/csv_config.yaml`
  ```yaml
  csv:
    text_column: "text"  # Nom de la colonne contenant le texte
    encoding: "auto"     # auto | utf-8 | latin-1 | ...
    delimiter: ","       # Séparateur CSV
    meta_columns: []     # Si vide, toutes colonnes sauf text_column
    skip_empty: true     # Ignorer les lignes avec texte vide
  ```

**Livrables** :
- Module `ingestion/csv_ingestion.py` fonctionnel
- Tests unitaires pour validations
- Documentation des paramètres

---

## Phase 3 : Bypass OCR et contrôleur de flux (Jour 2-3)

### 3.1 Factory/Router d'ingestion

**Objectif** : Centraliser la logique de choix de source

**Actions** :
- [ ] Créer `ingestion/ingestion_factory.py`
  - Fonction : `create_ingestion_pipeline(source_type: str, **kwargs) -> IngestionPipeline`
  - Classes :
    - `PDFIngestionPipeline` (existant, refactorisé)
    - `CSVIngestionPipeline` (nouveau)
    - Interface abstraite : `BaseIngestionPipeline` avec méthode `ingest() -> List[dict]`

- [ ] Refactoriser le code OCR existant
  - Extraire la logique OCR dans `PDFIngestionPipeline`
  - S'assurer que la sortie respecte `{"texteocr": str, "meta": dict}`
  - Éviter toute dépendance directe à PDF en aval

### 3.2 Centralisation de la construction de `texteocr`

**Objectif** : Un seul point de vérité pour `texteocr`

**Actions** :
- [ ] Créer `core/document.py`
  - Classe `Document` :
    ```python
    @dataclass
    class Document:
        texteocr: str
        meta: dict

        def validate(self):
            assert self.texteocr is not None
            assert isinstance(self.meta, dict)
    ```

- [ ] Modifier toutes les sources pour retourner des `Document`
  - PDF/OCR → `Document(texteocr=..., meta={...})`
  - CSV → `Document(texteocr=row[text_col], meta={autres colonnes})`

**Livrables** :
- Factory fonctionnel avec tests d'intégration
- Classe `Document` standardisée
- Disparition des références directes à PDF en dehors du module OCR

---

## Phase 4 : Gestion métadonnées génériques (Jour 3-4)

### 4.1 Refactorisation du chunking/indexation

**Objectif** : Accepter des métadonnées arbitraires

**Actions** :
- [ ] Auditer `scripts/rad_chunk.py`
  - Identifier tous les accès aux métadonnées (ex: `meta["filename"]`)
  - Remplacer les accès directs par `meta.get("filename", default)`
  - Ne plus supposer l'existence de champs spécifiques

- [ ] Enrichissement automatique des métadonnées
  - Pour CSV : ajouter `meta["source_type"] = "csv"`, `meta["row_index"] = i`
  - Pour PDF : conserver `meta["source_type"] = "pdf"`, `meta["page"] = ...`
  - Ajouter horodatage : `meta["ingested_at"] = datetime.now().isoformat()`

### 4.2 Transformation dynamique des colonnes CSV

**Objectif** : Mapper n'importe quelle colonne CSV vers `meta`

**Actions** :
- [ ] Dans `csv_ingestion.py`, implémenter :
  ```python
  def csv_row_to_document(row: pd.Series, text_column: str) -> Document:
      texteocr = str(row[text_column])
      meta = row.drop(text_column).to_dict()
      # Nettoyage : convertir NaN, dates, etc.
      meta = {k: sanitize_value(v) for k, v in meta.items()}
      return Document(texteocr, meta)
  ```

- [ ] Gérer les types de données
  - Convertir dates → ISO strings
  - Convertir NaN/None → `None` ou chaîne vide
  - Convertir booléens/nombres → types natifs Python

**Livrables** :
- Chunking agnostique de la source
- Tests avec méta variées (CSV 5 colonnes, CSV 20 colonnes, PDF)

---

## Phase 5 : Adaptations stockage/indexation (Jour 4-5)

### 5.1 Vectorisation générique

**Objectif** : Injecter dynamiquement toutes les métadonnées

**Actions** :
- [ ] Auditer les scripts de vectorisation
  - Identifier où les embeddings sont créés
  - Vérifier comment `meta` est passé au store vectoriel

- [ ] Assurer l'injection complète de `meta`
  - Si Pinecone : `upsert(vectors, metadata=dict(doc.meta))`
  - Supprimer toute whitelist de champs
  - Logger les clés de métadonnées envoyées

### 5.2 Mise à jour des connecteurs

**Objectif** : Compatibilité avec schéma dynamique

**Actions** :
- [ ] Si Pinecone/autre impose des contraintes
  - Vérifier les limites (taille des valeurs, types autorisés)
  - Implémenter un filtre : `filter_metadata_for_storage(meta) -> dict`
  - Logger les champs exclus si nécessaire

- [ ] Tester la recherche
  - Vérifier que les métadonnées CSV sont retrouvables
  - Tester des requêtes avec filtres sur champs CSV

**Livrables** :
- Pipeline vectorisation fonctionnel avec CSV
- Tests de bout en bout (ingestion CSV → recherche vectorielle)

---

## Phase 6 : Tests et validation (Jour 5-6)

### 6.1 Jeux de tests

**Objectif** : Garantir la robustesse

**Actions** :
- [ ] Créer `tests/fixtures/`
  - `simple.csv` : 2 colonnes (text, category)
  - `complex.csv` : 10+ colonnes avec types variés
  - `empty.csv` : Fichier vide
  - `malformed.csv` : Colonnes manquantes, encodage mixte

- [ ] Tests unitaires (`tests/test_csv_ingestion.py`)
  - Test ingestion CSV valide
  - Test CSV sans colonne texte → erreur
  - Test CSV vide → warning, pas d'erreur fatale
  - Test encodages variés (UTF-8, Latin-1)
  - Test métadonnées complexes (dates, nulls)

- [ ] Tests fonctionnels (`tests/test_pipeline_integration.py`)
  - Test flux complet : CSV → chunking → vectorisation
  - Test mélange PDF + CSV dans la même session
  - Test recherche avec filtres sur métadonnées CSV

- [ ] Tests de non-régression
  - Vérifier que le flux PDF/OCR existant fonctionne toujours
  - Comparer les résultats avant/après refactorisation

**Livrables** :
- Couverture de test > 80% sur le nouveau code
- Suite de tests CI/CD prête

---

## Phase 7 : Observabilité et logging (Jour 6)

### 7.1 Traçabilité du flux

**Objectif** : Déboguer facilement les ingestions

**Actions** :
- [ ] Enrichir `scripts/pdf_processing.log` (ou créer `ingestion.log`)
  - Logger le mode d'ingestion : `INFO: Ingestion mode: CSV`
  - Logger les métadonnées détectées : `INFO: Detected meta keys: [col1, col2, ...]`
  - Logger les étapes sautées : `INFO: Skipping OCR (texteocr provided)`

- [ ] Implémenter des métriques
  - Nombre de documents ingérés par source
  - Temps d'exécution par étape
  - Erreurs rencontrées (format structuré)

- [ ] Ajouter des logs de debug
  - Contenu des 100 premiers caractères de `texteocr`
  - Échantillon des métadonnées (sans données sensibles)

**Livrables** :
- Logs clairs permettant de suivre une ingestion CSV
- Dashboard ou script d'analyse des logs (bonus)

---

## Phase 8 : Documentation et DX (Jour 7)

### 8.1 Mise à jour de la documentation

**Objectif** : Faciliter l'adoption

**Actions** :
- [ ] Mettre à jour `README.md`
  - Ajouter section "Ingestion CSV"
  - Exemples de commandes :
    ```bash
    # Ingestion CSV
    python scripts/rad_chunk.py --source-type csv --input data/mon_fichier.csv

    # Ingestion PDF (par défaut)
    python scripts/rad_chunk.py --input data/document.pdf
    ```

- [ ] Créer `docs/csv_ingestion_guide.md`
  - Format CSV requis
  - Configuration des colonnes
  - Exemples de métadonnées
  - FAQ et troubleshooting

- [ ] Mettre à jour `AGENTS.md` (si applicable)
  - Décrire le nouveau workflow
  - Cas d'usage typiques (bases de connaissances tabulaires)

### 8.2 Variables d'environnement

**Objectif** : Clarifier la configuration

**Actions** :
- [ ] Créer `.env.example`
  ```
  RAGPY_SOURCE_TYPE=auto  # auto | pdf | csv
  RAGPY_CSV_TEXT_COLUMN=text
  RAGPY_CSV_ENCODING=utf-8
  ```

- [ ] Documenter dans `README.md` la hiérarchie de configuration :
  - Arguments CLI > Variables d'env > Fichier config > Défauts

**Livrables** :
- Documentation complète et accessible
- Exemples fonctionnels reproductibles

---

## Phase 9 : Déploiement et recette (Jour 8)

### 9.1 Préparation au déploiement

**Objectif** : Migrer en production sans casse

**Actions** :
- [ ] Vérifier les dépendances Python
  - Ajouter dans `requirements.txt` : `pandas`, `chardet` (si manquants)

- [ ] Préparer les migrations
  - Si base vectorielle : vérifier compatibilité avec nouveaux champs `meta`
  - Script de migration si nécessaire

- [ ] Créer un guide de rollback
  - Documenter comment revenir à la version précédente
  - Identifier les points de non-retour

### 9.2 Recette en sandbox

**Objectif** : Valider avant la prod

**Actions** :
- [ ] Déployer sur environnement de test
- [ ] Exécuter la suite de tests complète
- [ ] Ingérer un vrai CSV de production (données anonymisées)
- [ ] Valider les recherches vectorielles
- [ ] Mesurer les performances (temps d'exécution, RAM)

### 9.3 Intégration continue

**Objectif** : Automatiser les validations

**Actions** :
- [ ] Ajouter les tests CSV dans le pipeline CI/CD
- [ ] Configurer les alertes en cas d'échec
- [ ] Mettre à jour les hooks de pre-commit si nécessaire

**Livrables** :
- Environnement de staging validé
- Pipeline CI/CD à jour
- Go/No-go pour la production

---

## Risques et mitigation

| Risque | Probabilité | Impact | Mitigation |
|--------|-------------|--------|------------|
| Métadonnées CSV trop volumineuses pour le store vectoriel | Moyenne | Moyen | Implémenter un filtre de taille/type + warning |
| Régression sur pipeline PDF existant | Faible | Élevé | Tests de non-régression systématiques |
| CSV mal encodés causent des crashs | Élevée | Faible | Détection d'encodage robuste + fallback |
| Colonnes CSV avec noms invalides (espaces, caractères spéciaux) | Moyenne | Faible | Sanitisation des noms de colonnes |
| Performances dégradées avec gros CSV | Moyenne | Moyen | Traitement par batch + streaming si nécessaire |

---

## Métriques de succès

- [ ] Ingestion d'un CSV de 1000 lignes en < 2 minutes
- [ ] Aucune régression sur le pipeline PDF (tests passent)
- [ ] Recherche vectorielle fonctionne avec filtres sur métadonnées CSV
- [ ] Documentation lue et comprise par un nouvel utilisateur en < 10 min
- [ ] Couverture de tests > 80% sur le nouveau code

---

## Prochaines étapes (post-MVP)

- Support de fichiers Excel (`.xlsx`, `.xls`)
- Support de JSON/JSONL
- Interface web pour uploader des CSV
- Validation de schéma avancée (types, contraintes)
- Preview des données avant ingestion
- Support de CSV compressés (`.csv.gz`)

---

**Date de création** : 2025-10-21
**Statut** : Draft
**Responsable** : À définir
**Durée estimée** : 8 jours (1 développeur)
