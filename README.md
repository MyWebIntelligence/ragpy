# RAGpy: Pipeline de Traitement de Données & Interface Web

Ce dépôt fournit une **interface en ligne de commande (CLI)** et une **application web** pour traiter des documents (PDFs, exports Zotero) en chunks de texte avec des embeddings vectoriels (denses et sparse), et les charger dans une base de données vectorielle (Pinecone, Weaviate, ou Qdrant) pour des applications de Génération Augmentée par Récupération (RAG).

---

## Table des Matières

1.  [Prérequis](#prérequis)
2.  [Configuration (`.env`)](#configuration-env)
3.  [Architecture Générale du Processus](#architecture-générale-du-processus)
4.  [Détail du Processus (Scripts Individuels)](#détail-du-processus-scripts-individuels)
    *   [4.1 `rad_dataframe.py` (Extraction et Structuration)](#rad_dataframepy-extraction-et-structuration)
    *   [4.2 `rad_chunk.py` (Découpage, Nettoyage, Embeddings)](#rad_chunkpy-découpage-nettoyage-embeddings)
    *   [4.3 `rad_vectordb.py` (Chargement en Base Vectorielle)](#rad_vectordbpy-chargement-en-base-vectorielle)
5.  [Interface Web (Application FastAPI)](#interface-web-application-fastapi)
    *   [5.1 Démarrage du Serveur](#démarrage-du-serveur)
    *   [5.2 Flux de Travail dans l'Interface](#flux-de-travail-dans-linterface)
    *   [5.3 Gestion des Identifiants](#gestion-des-identifiants)
    *   [5.4 Autres Fonctionnalités](#autres-fonctionnalités)
6.  [Interface en Ligne de Commande (CLI)](#interface-en-ligne-de-commande-cli)
7.  [Structure du Projet](#structure-du-projet)
8.  [Dépannage et Conseils](#dépannage-et-conseils)

---

## Prérequis

-   **Python** 3.8 ou supérieur
-   **pip** (installeur de paquets Python)
-   **Git** (pour cloner ce dépôt)

Installez les dépendances principales :

```bash
pip install --upgrade pip
pip install -r scripts/requirements.txt
pip install fastapi uvicorn jinja2 python-multipart # Pour l'interface web
```

Le fichier `scripts/requirements.txt` inclut les paquets pour le traitement PDF, les embeddings, et les bases de données vectorielles :
-   pandas, pymupdf, tqdm, python-dateutil
-   openai, langchain-text-splitters, spacy
-   pinecone-client, weaviate-client, qdrant-client
-   python-dotenv

Pour le modèle spaCy français (si vous traitez du texte en français) :

```bash
python -m spacy download fr_core_news_md
```

---

## Configuration (`.env`)

1.  Copiez l'exemple :
    ```bash
    cp scripts/.env.example .env
    # Note: Le fichier .env doit être à la racine du projet ragpy/
    ```
    Si vous avez cloné le projet et que `.env.example` est dans `ragpy/scripts/`, ajustez la commande `cp` en conséquence ou déplacez le `.env` créé à la racine `ragpy/`. L'application web s'attend à trouver `.env` dans `ragpy/.env`.

2.  Éditez `ragpy/.env` et définissez vos identifiants :

    ```env
    OPENAI_API_KEY=votre_clé_api_openai
    PINECONE_API_KEY=votre_clé_api_pinecone
    # PINECONE_ENV est souvent géré par la librairie client ou non nécessaire pour les index serverless.
    WEAVIATE_CLUSTER_URL=votre_url_weaviate
    WEAVIATE_API_KEY=votre_clé_api_weaviate
    QDRANT_URL=votre_url_qdrant
    QDRANT_API_KEY=votre_clé_api_qdrant # optionnel pour les instances locales non sécurisées
    ```

Les scripts et l'application web chargeront ces variables automatiquement. Si une clé est manquante à l'exécution (pour les scripts CLI ou certaines opérations des scripts appelés par l'UI), vous pourriez être invité à la saisir interactivement. L'interface web fournit une section "Settings" pour gérer ces clés via le fichier `.env`.

---

## Architecture Générale du Processus

L'application `ragpy` transforme des documents bruts en données structurées et enrichies, prêtes à être utilisées dans des systèmes RAG. Le flux général est le suivant :

1.  **Entrée Utilisateur** :
    *   Via l'**interface web** : Un fichier ZIP contenant des documents (souvent un export Zotero avec un fichier JSON et des PDFs associés).
    *   Via la **CLI** : Un fichier JSON Zotero et un répertoire contenant les PDFs.
2.  **Extraction et Structuration (`rad_dataframe.py`)** :
    *   Le script analyse le JSON Zotero et les PDFs liés.
    *   Il extrait les métadonnées (titre, auteurs, date, etc.).
    *   Il extrait le texte brut des PDFs, en utilisant l'OCR (Optical Character Recognition) via PyMuPDF si nécessaire (pour les PDFs scannés ou à faible contenu textuel).
    *   Le résultat est un fichier CSV (`output.csv` dans un dossier de session unique pour l'UI web) qui structure ces informations.
3.  **Découpage, Nettoyage et Enrichissement (`rad_chunk.py`)** : Ce script opère en plusieurs phases :
    *   **Phase `initial`** :
        *   Lit le fichier CSV.
        *   Découpe le texte extrait (`texteocr`) en chunks plus petits en utilisant `RecursiveCharacterTextSplitter` de Langchain.
        *   Chaque chunk est ensuite "recodé" à l'aide d'un modèle LLM (ex: `gpt-4o-mini`) pour nettoyer le bruit d'OCR et améliorer la lisibilité, tout en préservant le contenu sémantique.
        *   Produit un fichier JSON (`output_chunks.json`) contenant les chunks nettoyés et leurs métadonnées.
    *   **Phase `dense`** :
        *   Prend en entrée `output_chunks.json`.
        *   Génère des embeddings vectoriels denses pour chaque chunk de texte en utilisant un modèle d'embedding OpenAI (ex: `text-embedding-3-large`).
        *   Produit `output_chunks_with_embeddings.json`, ajoutant les vecteurs denses aux données des chunks.
    *   **Phase `sparse`** :
        *   Prend en entrée `output_chunks_with_embeddings.json`.
        *   Utilise `spaCy` pour l'analyse linguistique (lemmatisation, filtrage des mots non pertinents).
        *   Génère des embeddings vectoriels sparse (basés sur la fréquence des termes - TF) pour chaque chunk.
        *   Produit `output_chunks_with_embeddings_sparse.json`, ajoutant les vecteurs sparse.
4.  **Chargement en Base Vectorielle (`rad_vectordb.py`)** :
    *   Prend en entrée `output_chunks_with_embeddings_sparse.json`.
    *   Permet à l'utilisateur de choisir une base de données vectorielle cible (Pinecone, Weaviate, Qdrant).
    *   Prépare les données (vecteurs denses, vecteurs sparse, métadonnées) au format spécifique de la base choisie.
    *   Se connecte à la base de données et y insère les données par lots.
5.  **Sortie** : Les chunks et leurs embeddings sont stockés dans la base vectorielle choisie, prêts pour des requêtes de similarité dans un système RAG.

L'interface web guide l'utilisateur à travers ces étapes de manière interactive, tandis que la CLI permet d'automatiser le processus.

---

## Détail du Processus (Scripts Individuels)

### 4.1 `rad_dataframe.py` (Extraction et Structuration)

Ce script est la première étape du pipeline, transformant les données brutes Zotero (ou un dossier de PDFs) en un format tabulaire structuré.

-   **Entrée** :
    -   Un fichier JSON exporté de Zotero (argument `--json`).
    -   Un répertoire de base (argument `--dir`) où se trouvent les fichiers PDF référencés dans le JSON.
-   **Logique Principale** :
    1.  Charge les métadonnées des items depuis le fichier JSON Zotero.
    2.  Pour chaque item, il recherche les fichiers PDF attachés.
    3.  Les chemins relatifs des PDF sont résolus en utilisant le `--dir` fourni.
    4.  Utilise la bibliothèque `PyMuPDF (fitz)` pour ouvrir et lire chaque fichier PDF.
    5.  **Extraction de Texte** :
        *   Tente une extraction de texte standard (`page.get_text("text")`).
        *   Si le texte extrait est court (heuristique : moins de 50 mots), cela suggère un PDF basé sur des images ou un scan. Dans ce cas, une extraction OCR est tentée (`page.get_text("ocr")`).
    6.  **Extraction des Métadonnées du PDF** : Titre, auteur, date de création, et DOI (tentative d'extraction à partir des métadonnées du PDF ou du texte des premières pages).
    7.  Toutes les informations (métadonnées Zotero, métadonnées PDF, texte extrait `texteocr`) sont compilées dans un DataFrame Pandas.
-   **Sortie** :
    -   Un fichier CSV (spécifié par `--output`) contenant une ligne par document PDF traité, avec ses métadonnées et le texte intégral extrait. Dans l'interface web, ce fichier est nommé `output.csv` et est sauvegardé dans le dossier de session de l'upload.
-   **Journalisation** : Les opérations et erreurs sont enregistrées dans `ragpy/logs/pdf_processing.log`.

### 4.2 `rad_chunk.py` (Découpage, Nettoyage, Embeddings)

Ce script prend le CSV généré et le transforme en chunks de texte enrichis avec des embeddings. Il opère en plusieurs phases distinctes, qui peuvent être appelées séquentiellement par l'interface web ou en une seule fois (`--phase all`) par la CLI.

-   **Configuration Initiale** :
    -   Charge la clé API OpenAI depuis `.env` (ou la demande à l'utilisateur).
    -   Initialise le client OpenAI.
    -   Initialise `RecursiveCharacterTextSplitter` de `langchain_text_splitters` (modèle `text-embedding-3-large` pour le comptage de tokens, taille de chunk de 8000, chevauchement de 800).
    -   Charge le modèle spaCy `fr_core_news_md` (le télécharge si absent).

-   **Phase `initial` : Découpage et Recodage GPT**
    -   **Entrée** : Fichier CSV (généré par `rad_dataframe.py`).
    -   **Logique** :
        1.  Lit le CSV et itère sur chaque document (ligne).
        2.  Le champ `texteocr` est découpé en chunks plus petits par `TEXT_SPLITTER`.
        3.  **Recodage GPT** : Chaque chunk brut est envoyé à `gpt-4o-mini` (par lots) avec des instructions spécifiques : "Nettoie ce chunk pour en faire un texte propre qui commence par une phrase complète et se termine par un point. Supprime le bruit d'OCR et les imperfections en conservant le sens original. Ne change ni ajoute aucun mot du texte d'origine."
        4.  Les chunks nettoyés, ainsi que les métadonnées héritées du document original, sont sauvegardés de manière incrémentale.
    -   **Sortie** : Un fichier JSON (ex: `output_chunks.json`) contenant une liste d'objets, chaque objet représentant un chunk avec son texte nettoyé et ses métadonnées (ID, titre du document, auteurs, date, etc.).

-   **Phase `dense` : Génération des Embeddings Denses**
    -   **Entrée** : Fichier JSON des chunks nettoyés (ex: `output_chunks.json`).
    -   **Logique** :
        1.  Charge les chunks depuis le fichier JSON.
        2.  Pour chaque chunk, le `chunk_text` est envoyé à l'API OpenAI pour générer un embedding dense en utilisant le modèle `text-embedding-3-large`.
        3.  Les embeddings sont calculés par lots pour optimiser les appels API.
    -   **Sortie** : Un nouveau fichier JSON (ex: `output_chunks_with_embeddings.json`) où chaque objet chunk inclut maintenant une clé `embedding` avec le vecteur dense.

-   **Phase `sparse` : Génération des Embeddings Sparse**
    -   **Entrée** : Fichier JSON des chunks avec embeddings denses (ex: `output_chunks_with_embeddings.json`).
    -   **Logique** :
        1.  Charge les chunks.
        2.  Pour chaque `chunk_text` :
            *   Le texte est traité par `spaCy` (modèle `fr_core_news_md`) pour la tokenisation et la lemmatisation.
            *   Les lemmes des mots pertinents (Noms, Adjectifs, Verbes, Noms Propres), en excluant les mots vides et la ponctuation, sont extraits.
            *   Une représentation sparse (dictionnaire d'indices et de valeurs) est créée basée sur la fréquence (TF - Term Frequency) de ces lemmes. Les indices sont générés par un hachage du lemme.
    -   **Sortie** : Un fichier JSON final (ex: `output_chunks_with_embeddings_sparse.json`) où chaque objet chunk inclut maintenant une clé `sparse_embedding` (contenant `indices` et `values`), en plus de l'embedding dense.

### 4.3 `rad_vectordb.py` (Chargement en Base Vectorielle)

Ce script est la dernière étape, chargeant les chunks enrichis dans une base de données vectorielle.

-   **Entrée** : Fichier JSON contenant les chunks avec leurs embeddings denses et sparse (ex: `output_chunks_with_embeddings_sparse.json`).
-   **Logique Générale** :
    1.  Charge les chunks depuis le fichier JSON.
    2.  Demande à l'utilisateur (en CLI) ou reçoit de l'interface web le choix de la base de données (Pinecone, Weaviate, Qdrant) et les informations de connexion/indexation nécessaires (nom d'index, de classe, de collection, URL, clés API depuis `.env`).
    3.  Pour chaque base de données :
        *   **Préparation des Données** : Les chunks sont transformés au format spécifique attendu par la base de données cible. Cela inclut le formatage des vecteurs denses, des vecteurs sparse (si la base les supporte nativement de cette manière), et des métadonnées. Des UUIDs stables sont souvent générés pour les IDs des vecteurs.
        *   **Connexion** : Établit une connexion à l'instance de la base de données.
        *   **Gestion de l'Index/Collection** : Vérifie si l'index/collection cible existe. Pour Qdrant, il peut la créer si elle est absente, en déduisant la dimension du vecteur du premier chunk. Pour Weaviate, il gère les tenants. Pour Pinecone, l'index doit généralement préexister.
        *   **Insertion par Lots** : Les données sont insérées (upserted) par lots pour des raisons de performance et pour respecter les limites des API. Des mécanismes de relance simples sont implémentés en cas d'erreurs transitoires.
-   **Bases de Données Supportées** :
    *   **Pinecone** : Supporte les vecteurs denses et sparse. Le script prépare les `vectors` avec `id`, `values` (dense), `sparse_values` (si présents dans le JSON), et `metadata`.
    *   **Weaviate** : Le script utilise `insert_to_weaviate_hybrid`. Il crée des `DataObject` avec les `properties` (métadonnées), un `uuid` généré, et le `vector` (dense). Gère la création de tenants. Les dates sont normalisées au format RFC3339.
    *   **Qdrant** : Crée des `PointStruct` avec un `id` (UUID généré), `vector` (dense), et `payload` (métadonnées). Peut créer la collection si elle n'existe pas, en utilisant la distance Cosine par défaut.
-   **Sortie** : Les données sont stockées dans la base vectorielle choisie. Le script affiche des messages sur le statut de l'insertion.

---

## Interface Web (Application FastAPI)

L'application web, servie par FastAPI, offre une manière interactive de piloter le pipeline de traitement. Le code principal se trouve dans `ragpy/app/main.py`.

### 5.1 Démarrage du Serveur

Depuis la racine du projet (`ragpy/`), exécutez :

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

L'option `--reload` permet au serveur de redémarrer automatiquement lors de modifications du code.

### 5.2 Flux de Travail dans l'Interface

Ouvrez votre navigateur à l'adresse [http://localhost:8000](http://localhost:8000).

1.  **Étape 1 : Téléchargement du ZIP (`POST /upload_zip`)**
    *   L'utilisateur sélectionne et télécharge un fichier ZIP. Ce ZIP contient généralement un export Zotero (un fichier `.json` et un dossier `files/` avec les PDFs) ou directement une collection de PDFs.
    *   **Serveur (`main.py`)** :
        *   Génère un préfixe unique (`uuid`) pour le nom du fichier ZIP afin d'éviter les conflits.
        *   Sauvegarde le ZIP dans `ragpy/uploads/<uuid>_<nom_original>.zip`.
        *   Extrait le contenu du ZIP dans un nouveau sous-dossier unique (ex: `ragpy/uploads/<uuid>_<nom_original>/`).
        *   Si le ZIP contient un seul dossier racine (cas fréquent pour les exports Zotero), le chemin de traitement est ajusté pour pointer directement dans ce dossier.
        *   Analyse l'arborescence des fichiers extraits.
        *   Retourne à l'interface le chemin relatif du dossier de traitement (par rapport à `ragpy/uploads/`) et l'arborescence des fichiers pour affichage.

2.  **Étape 2 : Traitement du DataFrame (`POST /process_dataframe`)**
    *   L'interface envoie le chemin relatif du dossier de traitement (obtenu à l'étape 1).
    *   **Serveur (`main.py`)** :
        *   Reconstruit le chemin absolu du dossier de traitement.
        *   Localise le premier fichier `.json` trouvé dans ce dossier (supposé être l'export Zotero).
        *   Exécute le script `ragpy/scripts/rad_dataframe.py` en tant que sous-processus. Les arguments `--json` (chemin du JSON trouvé), `--dir` (chemin du dossier de traitement), et `--output` (chemin pour `output.csv` dans le dossier de traitement) sont passés au script.
        *   Une fois le script terminé, le serveur lit les 5 premières lignes du fichier `output.csv` généré pour créer un aperçu.
        *   Retourne à l'interface le chemin du fichier `output.csv` et les données d'aperçu.

3.  **Étape 3 : Chunking et Génération des Embeddings (Appels Séquentiels)**
    L'interface déclenche séquentiellement les phases de `rad_chunk.py` via des endpoints dédiés. Chaque endpoint exécute `ragpy/scripts/rad_chunk.py` en sous-processus avec l'argument `--phase` approprié.
    *   **Initial Text Chunking (`POST /initial_text_chunking`)** :
        *   Entrée serveur : Chemin relatif du dossier de traitement.
        *   Logique serveur : Exécute `rad_chunk.py --phase initial --input <chemin_vers_output.csv> --output <dossier_de_traitement>`.
        *   Sortie attendue du script : `output_chunks.json` dans le dossier de traitement.
        *   Réponse au client : Statut, chemin du fichier JSON, et nombre de chunks.
    *   **Dense Embedding Generation (`POST /dense_embedding_generation`)** :
        *   Entrée serveur : Chemin relatif du dossier de traitement.
        *   Logique serveur : Exécute `rad_chunk.py --phase dense --input <chemin_vers_output_chunks.json> --output <dossier_de_traitement>`.
        *   Sortie attendue du script : `output_chunks_with_embeddings.json`.
        *   Réponse au client : Statut, chemin du fichier JSON, et nombre de chunks.
    *   **Sparse Embedding Generation (`POST /sparse_embedding_generation`)** :
        *   Entrée serveur : Chemin relatif du dossier de traitement.
        *   Logique serveur : Exécute `rad_chunk.py --phase sparse --input <chemin_vers_output_chunks_with_embeddings.json> --output <dossier_de_traitement>`.
        *   Sortie attendue du script : `output_chunks_with_embeddings_sparse.json`.
        *   Réponse au client : Statut, chemin du fichier JSON, et nombre de chunks.
    *   **Aperçu des Chunks (`GET /get_first_chunk`)** : Après chaque phase, l'interface peut appeler cet endpoint pour afficher un aperçu du premier chunk du fichier JSON généré (texte et extrait des embeddings).

4.  **Étape 4 : Téléchargement vers la Base de Données Vectorielle (`POST /upload_db`)**
    *   L'utilisateur sélectionne la base de données (Pinecone, Weaviate, Qdrant) dans l'interface et fournit les informations spécifiques (nom d'index/classe/collection, nom de tenant pour Weaviate).
    *   **Serveur (`main.py`)** :
        *   Récupère les identifiants API nécessaires depuis `ragpy/.env` (gérés via la section "Settings" de l'UI).
        *   Appelle la fonction d'insertion appropriée du script `ragpy/scripts/rad_vectordb.py` (ex: `insert_to_pinecone`, `insert_to_weaviate_hybrid`, `insert_to_qdrant`). L'entrée pour ces fonctions est le fichier `output_chunks_with_embeddings_sparse.json` du dossier de traitement.
        *   Retourne un message de statut sur le succès ou l'échec de l'opération d'upload.

### 5.3 Gestion des Identifiants

L'interface web inclut une section "Settings" (accessible via l'icône d'engrenage) qui interagit avec les endpoints suivants pour gérer le fichier `ragpy/.env` :

-   **`GET /get_credentials`** : Lit le fichier `ragpy/.env` et retourne les valeurs actuelles des clés API (OpenAI, Pinecone, Weaviate, Qdrant) pour peupler le formulaire dans les paramètres.
-   **`POST /save_credentials`** : Reçoit les nouvelles valeurs du formulaire et met à jour (ou crée) le fichier `ragpy/.env` avec ces informations.

### 5.4 Autres Fonctionnalités

-   **Téléchargement de Fichiers (`GET /download_file`)** : Permet à l'utilisateur de télécharger les fichiers intermédiaires ou finaux générés pendant le processus (ex: `output.csv`, `output_chunks_with_embeddings_sparse.json`) à partir du dossier de session.
-   **Arrêt des Scripts (`POST /stop_all_scripts`)** : Tente d'arrêter tous les scripts `rad_*.py` en cours d'exécution en envoyant un signal `SIGTERM` via la commande `pkill`. Utile si un processus est bloqué ou prend trop de temps.

---

## Interface en Ligne de Commande (CLI)

Pour ceux qui préfèrent ou ont besoin d'automatiser le pipeline, les scripts peuvent être exécutés directement depuis la ligne de commande. Assurez-vous que votre fichier `.env` est correctement configuré à la racine `ragpy/`.

1.  **Extraction de texte & métadonnées (`rad_dataframe.py`)**
    Exemple pour un export Zotero nommé `MaBiblio` situé dans `ragpy/sources/MaBiblio/` et contenant `MaBiblio.json` et les PDFs :
    ```bash
    python scripts/rad_dataframe.py \
      --json sources/MaBiblio/MaBiblio.json \
      --dir sources/MaBiblio \
      --output sources/MaBiblio/output.csv
    ```
    -   `--json`: Chemin vers le fichier JSON Zotero.
    -   `--dir`: Répertoire de base pour les fichiers PDF référencés dans le JSON.
    -   `--output`: Chemin du fichier CSV de sortie pour les métadonnées et le texte OCR.

2.  **Découpage du texte et génération des embeddings (`rad_chunk.py`)**
    En supposant que le CSV de l'étape 1 est `sources/MaBiblio/output.csv` et que les sorties JSON doivent aller dans `sources/MaBiblio/` :
    ```bash
    python scripts/rad_chunk.py \
      --input sources/MaBiblio/output.csv \
      --output sources/MaBiblio \
      --phase all
    ```
    -   `--input`: Fichier CSV généré à l'étape 1.
    -   `--output`: Répertoire où sauvegarder les fichiers JSON intermédiaires et finaux :
        -   `output_chunks.json`
        -   `output_chunks_with_embeddings.json`
        -   `output_chunks_with_embeddings_sparse.json` (le nom de base `output` est dérivé du nom du fichier CSV d'entrée si possible, sinon il utilise "output" par défaut).
    -   `--phase all`: Exécute toutes les phases (initial, dense, sparse). D'autres options sont `initial`, `dense`, `sparse`.

3.  **Téléchargement vers la base de données vectorielle (`rad_vectordb.py`)**
    Ce script est interactif en CLI. Il vous demandera :
    ```bash
    python scripts/rad_vectordb.py
    ```
    1.  Le choix de la base de données (1: Pinecone, 2: Weaviate, 3: Qdrant).
    2.  Le chemin vers le fichier JSON des embeddings (ex: `sources/MaBiblio/output_chunks_with_embeddings_sparse.json`).
    3.  Les informations spécifiques à la base de données (nom d'index/collection, etc.). Les clés API sont lues depuis `.env`.

---

## Structure du Projet

```
ragpy/
├── .env                  # Fichier de configuration des clés API (à créer depuis .env.example)
├── app/                  # Code de l'application web FastAPI
│   ├── main.py           # Logique principale de l'API et des routes web
│   ├── static/           # Fichiers statiques (CSS, JS, images pour l'UI)
│   │   └── style.css
│   │   └── favicon.ico
│   └── templates/        # Modèles HTML Jinja2 pour l'interface web
│       └── index.html
├── logs/                 # Répertoire pour les fichiers journaux
│   ├── app.log           # Logs de l'application web FastAPI
│   └── pdf_processing.log # Logs spécifiques au script rad_dataframe.py
├── scripts/              # Scripts Python pour le pipeline de traitement de données
│   ├── .env.example      # Modèle pour le fichier .env
│   ├── rad_chunk.py      # Découpage, nettoyage GPT, embeddings denses et sparse
│   ├── rad_dataframe.py  # Extraction depuis Zotero/PDF vers CSV
│   ├── rad_vectordb.py   # Chargement vers les bases vectorielles
│   └── requirements.txt  # Dépendances Python pour les scripts
├── uploads/              # Répertoire où les fichiers ZIP sont téléchargés et traités par l'UI web
│                           # (Contient des sous-dossiers par session d'upload)
└── README.md             # Ce fichier
```

---

## Dépannage et Conseils

-   **Clés API manquantes**: Vérifiez que votre fichier `ragpy/.env` existe à la racine du dossier `ragpy/` et que les valeurs sont correctes. L'interface web (section "Settings") permet de les gérer.
-   **Dépendances**: Assurez-vous que tous les paquets listés dans `scripts/requirements.txt`, ainsi que `fastapi` et `uvicorn`, sont installés dans votre environnement Python.
-   **Chemins de fichiers**: Utilisez des chemins absolus ou correctement relatifs lors de l'utilisation de la CLI. L'interface web gère les chemins relatifs au sein du dossier `uploads/`.
-   **Modèle spaCy**: Si `rad_chunk.py` signale un modèle linguistique manquant, exécutez `python -m spacy download fr_core_news_md`.
-   **Configuration de la base de données vectorielle**:
    -   **Pinecone**: Créez au préalable un index avec la dimension vectorielle correspondante au modèle d'embedding utilisé (ex: 1536 pour `text-embedding-ada-002`, 3072 pour `text-embedding-3-large`).
    -   **Weaviate**: Définissez une classe et un schéma dans votre cluster. Le script gère la création de tenants et l'insertion des données.
    -   **Qdrant**: Fournissez l'URL et optionnellement la clé API. Le script tentera de créer la collection si elle n'existe pas.
-   **Performance**: Le traitement de gros volumes de documents et la génération d'embeddings peuvent prendre du temps. Surveillez les logs (`ragpy/logs/app.log`, `ragpy/logs/pdf_processing.log`) et la console du serveur `uvicorn` pour suivre la progression. L'interface web fournit également des mises à jour de statut.
-   **Erreurs de script en sous-processus** : Si l'interface web signale une erreur lors de l'exécution d'un script, consultez `ragpy/logs/app.log` pour les détails de la commande exécutée et les éventuels messages d'erreur du script lui-même.

---

_Fin du guide._
