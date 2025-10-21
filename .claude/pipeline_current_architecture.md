# Architecture actuelle du pipeline RAGpy

**Date de création** : 2025-10-21
**Objectif** : Documenter l'architecture existante pour préparer l'ingestion CSV

---

## Vue d'ensemble du flux de données

```
┌─────────────────────────────────────────────────────────────────┐
│                      PIPELINE ACTUEL (PDF-only)                  │
└─────────────────────────────────────────────────────────────────┘

1. rad_dataframe.py : Zotero JSON + PDF → CSV
   ├─ Input : Zotero JSON + répertoire PDF
   ├─ OCR   : Mistral → OpenAI → PyMuPDF (legacy)
   └─ Output: CSV avec colonnes fixes + texteocr + texteocr_provider

2. rad_chunk.py : CSV → JSON avec chunks + embeddings
   ├─ Phase initial : CSV → chunks JSON
   ├─ Phase dense   : chunks → chunks + embeddings denses
   └─ Phase sparse  : chunks → chunks + embeddings denses + sparses

3. rad_vectordb.py : JSON → Base vectorielle
   ├─ Pinecone  : upsert avec métadonnées hardcodées
   ├─ Weaviate  : insert avec métadonnées hardcodées
   └─ Qdrant    : upsert avec métadonnées hardcodées
```

---

## 1. Module `rad_dataframe.py` — Extraction PDF/OCR

### Responsabilités
- Charger un export Zotero (JSON)
- Localiser les PDF référencés (avec recherche fuzzy si nécessaire)
- Extraire le texte via OCR multi-provider
- Produire un CSV avec métadonnées + texte

### Fonctions clés

| Fonction | Localisation | Rôle |
|----------|-------------|------|
| `extract_text_with_ocr()` | ligne 337 | Point d'entrée OCR avec cascade Mistral → OpenAI → Legacy |
| `_extract_text_with_mistral()` | ligne 144 | OCR Mistral (retourne Markdown) |
| `_extract_text_with_openai()` | ligne 267 | OCR OpenAI vision (fallback) |
| `_extract_text_with_legacy_pdf()` | ligne 120 | OCR PyMuPDF (fallback final) |
| `load_zotero_to_dataframe()` | ligne 397 | Orchestration : JSON → DataFrame |

### Flux détaillé

```python
# Ligne 490-509 : Extraction OCR avec détails
ocr_payload = extract_text_with_ocr(
    actual_pdf_path,
    return_details=True,  # Retourne OCRResult(text, provider)
)

# Ligne 503-510 : Construction du record
records.append({
    # Métadonnées Zotero
    "type": ...,
    "title": ...,
    "authors": ...,
    "date": ...,
    "url": ...,
    "doi": ...,
    "filename": ...,
    "path": ...,
    "attachment_title": ...,

    # Texte OCR - POINT CRITIQUE
    "texteocr": ocr_payload.text,           # ← Source unique du texte
    "texteocr_provider": ocr_payload.provider  # ← "mistral" | "openai" | "legacy"
})
```

### Colonnes CSV produites (hardcodées)

| Colonne | Type | Description |
|---------|------|-------------|
| `type` | str | Type d'item Zotero ("article", "book", etc.) |
| `title` | str | Titre du document |
| `authors` | str | Auteurs (jointure par virgule) |
| `date` | str | Date de publication |
| `url` | str | URL de l'article |
| `doi` | str | Digital Object Identifier |
| `filename` | str | Nom du fichier PDF |
| `path` | str | Chemin complet résolu du PDF |
| `attachment_title` | str | Titre de l'attachement Zotero |
| **`texteocr`** | **str** | **Texte extrait par OCR (variable pivot)** |
| **`texteocr_provider`** | **str** | **Fournisseur OCR utilisé** |

### Variables d'environnement

```bash
# OCR Mistral (prioritaire)
MISTRAL_API_KEY
MISTRAL_API_BASE_URL="https://api.mistral.ai"
MISTRAL_OCR_MODEL="mistral-ocr-latest"
MISTRAL_OCR_TIMEOUT=300
MISTRAL_DELETE_UPLOADED_FILE=true

# OCR OpenAI (fallback)
OPENAI_API_KEY
OPENAI_OCR_MODEL="gpt-4o-mini"
OPENAI_OCR_PROMPT="Transcris cette page PDF en Markdown..."
OPENAI_OCR_MAX_PAGES=10
OPENAI_OCR_MAX_TOKENS=2048
OPENAI_OCR_RENDER_SCALE=2.0
```

---

## 2. Module `rad_chunk.py` — Chunking et embeddings

### Responsabilités
- Découper `texteocr` en chunks (~1000 tokens)
- Recoder les chunks via GPT (sauf si OCR Mistral)
- Générer embeddings denses (OpenAI `text-embedding-3-large`)
- Générer embeddings sparses (spaCy TF lemmatisé)

### Fonctions clés

| Fonction | Localisation | Rôle |
|----------|-------------|------|
| `process_document_chunks()` | ligne 187 | Orchestration chunking + recodage pour 1 document |
| `gpt_recode_batch()` | ligne 109 | Recodage GPT par lot (5 chunks par défaut) |
| `get_embeddings_batch()` | ligne 301 | Embeddings denses OpenAI |
| `extract_sparse_features()` | ligne 409 | Embeddings sparses spaCy |
| `save_raw_chunks_to_json_incrementally()` | ligne 169 | Sauvegarde thread-safe incrémentale |

### Flux de traitement (3 phases)

#### Phase `initial` : CSV → Chunks JSON

```python
# Ligne 199 : POINT D'ENTRÉE du texte
text = row_data.get("texteocr", "").strip()

# Ligne 204-208 : Décision de recodage selon provider
provider = str(row_data.get("texteocr_provider", "")).lower()
recode_required = provider != "mistral"  # Skip GPT si déjà Markdown

# Ligne 211 : Chunking
text_chunks = TEXT_SPLITTER.split_text(text)  # RecursiveCharacterTextSplitter

# Ligne 221-233 : Recodage conditionnel
if recode_required:
    cleaned_batch = gpt_recode_batch(
        batch_to_recode,
        instructions="Nettoie ce chunk OCR brut...",
        model="gpt-4o-mini"
    )
else:
    cleaned_batch = batch_to_recode  # Pas de recodage
```

#### Métadonnées des chunks (hardcodées !)

```python
# Ligne 250-263 : Construction de chunk_metadata
chunk_metadata = {
    "id":           f"{doc_id}_{original_chunk_index}",
    "type":         sanitize_metadata_value(row_data.get("type", "")),
    "title":        sanitize_metadata_value(row_data.get("title", "")),
    "authors":      sanitize_metadata_value(row_data.get("authors", "")),
    "date":         sanitize_metadata_value(row_data.get("date", "")),
    "filename":     sanitize_metadata_value(row_data.get("filename", "")),
    "doc_id":       doc_id,
    "chunk_index":  original_chunk_index,
    "total_chunks": len(text_chunks),
    "text":         cleaned_text,
    "ocr_provider": sanitize_metadata_value(provider, ""),
}
```

**Problème identifié** : Liste de champs **hardcodée** ! Toute colonne CSV non listée ici est **perdue**.

#### Phase `dense` : Chunks → Chunks + Embeddings denses

```python
# Ligne 301-318 : Embeddings denses OpenAI
def get_embeddings_batch(texts, model="text-embedding-3-large"):
    response = client.embeddings.create(input=texts, model=model)
    return [item.embedding for item in response.data]
```

- Traite les chunks par lots de 32 (configurable)
- Retries automatiques en cas d'erreur API
- Sauvegarde intermédiaire tous les ~1000 chunks

#### Phase `sparse` : Chunks → Chunks + Embeddings sparses

```python
# Ligne 409-449 : Embeddings sparses spaCy
def extract_sparse_features(text):
    doc = nlp(text)  # spaCy fr_core_news_md
    relevant_pos = {"NOUN", "PROPN", "ADJ", "VERB"}
    lemmas = [token.lemma_.lower() for token in doc
              if token.pos_ in relevant_pos
              and not token.is_stop
              and len(token.lemma_) > 1]

    # TF normalisé avec hachage mod 100k
    counts = Counter(lemmas)
    for lemma, count in counts.items():
        index = hash(lemma) % 100000
        sparse_dict[str(index)] = count / total_lemmas_in_doc

    return {"indices": [...], "values": [...]}
```

### Variables d'environnement

```bash
OPENAI_API_KEY  # Obligatoire pour embeddings + recodage
```

### Configurations internes

```python
DEFAULT_MAX_WORKERS = os.cpu_count() - 1  # Parallélisme général
DEFAULT_BATCH_SIZE_GPT = 5                # Recodage GPT
DEFAULT_EMBEDDING_BATCH_SIZE = 32         # Embeddings denses
TEXT_SPLITTER:
  - chunk_size: 1000 tokens
  - chunk_overlap: 150 tokens
  - separators: ["\n\n", "#", "##", "\n", " ", ""]
```

---

## 3. Module `rad_vectordb.py` — Insertion vectorielle

### Responsabilités
- Charger JSON avec embeddings (denses + sparses)
- Formatter les données pour chaque provider
- Upserter par lots vers Pinecone / Weaviate / Qdrant

### Fonctions clés par provider

#### Pinecone

| Fonction | Localisation | Rôle |
|----------|-------------|------|
| `prepare_vectors_for_pinecone()` | ligne 66 | Formatte chunks → vecteurs Pinecone |
| `upsert_batch_to_pinecone()` | ligne 29 | Upsert 1 lot avec retry |
| `insert_to_pinecone()` | ligne 123 | Orchestration complète |

```python
# Ligne 85-95 : MÉTADONNÉES HARDCODÉES
metadata = {
    "title": chunk.get("title", ""),
    "authors": chunk.get("authors", ""),
    "date": chunk.get("date", ""),
    "type": chunk.get("type", ""),
    "filename": chunk.get("filename", ""),
    "doc_id": chunk.get("doc_id", ""),
    "chunk_index": chunk.get("chunk_index", 0),
    "total_chunks": chunk.get("total_chunks", 0),
    "text": chunk.get("text") or chunk.get("chunk_text", "")
}
```

**Problème** : Métadonnées **hardcodées** ! Impossible d'injecter des colonnes CSV arbitraires.

#### Weaviate

| Fonction | Localisation | Rôle |
|----------|-------------|------|
| `insert_to_weaviate_hybrid()` | ligne 436 | Orchestration complète avec multi-tenancy |
| `generate_uuid()` | ligne 385 | UUID v5 stable |
| `normalize_date_to_rfc3339()` | ligne 399 | Conversion dates → RFC3339 |

```python
# Ligne 541-551 : MÉTADONNÉES HARDCODÉES (même liste)
properties = {
    "title": chunk.get("title", ""),
    "authors": chunk.get("authors", ""),
    "date": normalize_date_to_rfc3339(chunk.get("date", "")),
    "type": chunk.get("type", ""),
    "filename": chunk.get("filename", ""),
    "doc_id": chunk.get("doc_id", ""),
    "chunk_index": chunk.get("chunk_index", 0),
    "total_chunks": chunk.get("total_chunks", 0),
    "text": chunk.get("text") or chunk.get("chunk_text", "")
}
```

#### Qdrant

| Fonction | Localisation | Rôle |
|----------|-------------|------|
| `prepare_points_for_qdrant()` | ligne 604 | Formatte chunks → PointStruct |
| `upsert_batch_to_qdrant()` | ligne 661 | Upsert 1 lot avec retry |
| `insert_to_qdrant()` | ligne 709 | Orchestration complète |

```python
# Ligne 636-647 : MÉTADONNÉES HARDCODÉES (même liste)
payload = {
    "original_id": chunk["id"],
    "title": chunk.get("title", ""),
    "authors": chunk.get("authors", ""),
    "date": chunk.get("date", ""),
    "type": chunk.get("type", ""),
    "filename": chunk.get("filename", ""),
    "doc_id": chunk.get("doc_id", ""),
    "chunk_index": chunk.get("chunk_index", 0),
    "total_chunks": chunk.get("total_chunks", 0),
    "text": chunk.get("text") or chunk.get("chunk_text", "")
}
```

### Configurations

```python
PINECONE_BATCH_SIZE = 100
WEAVIATE_BATCH_SIZE = 100
QDRANT_BATCH_SIZE = 100
```

### Variables d'environnement

```bash
# Pinecone
PINECONE_API_KEY
PINECONE_ENV  # Optionnel selon SDK version

# Weaviate
WEAVIATE_URL
WEAVIATE_API_KEY

# Qdrant
QDRANT_URL
QDRANT_API_KEY  # Optionnel (instances locales)
```

---

## Points critiques identifiés pour l'ingestion CSV

### 1. Variable pivot unique : `texteocr`

| Point de création/consommation | Fichier | Ligne |
|-------------------------------|---------|-------|
| **Création (OCR)** | rad_dataframe.py | 508 |
| **Consommation (chunking)** | rad_chunk.py | 199 |

**Conclusion** : `texteocr` est le **seul point d'entrée** du contenu textuel dans le pipeline. Pour ingérer du CSV, il suffit de mapper une colonne CSV → `texteocr`.

### 2. Métadonnées hardcodées (3 emplacements)

| Emplacement | Fichier | Ligne | Impact |
|------------|---------|-------|--------|
| Création chunks | rad_chunk.py | 250-263 | Colonnes CSV non listées → perdues |
| Préparation Pinecone | rad_vectordb.py | 85-95 | Impossible d'ajouter champs CSV |
| Préparation Weaviate | rad_vectordb.py | 541-551 | Impossible d'ajouter champs CSV |
| Préparation Qdrant | rad_vectordb.py | 636-647 | Impossible d'ajouter champs CSV |

**Conclusion** : Les 3 connecteurs utilisent la **même liste hardcodée** de champs. Refactorisation nécessaire pour accepter des métadonnées dynamiques.

### 3. Logique de recodage GPT liée à `texteocr_provider`

```python
# rad_chunk.py:204-208
provider = str(row_data.get("texteocr_provider", "")).lower()
recode_required = provider != "mistral"
```

**Impact pour CSV** : Si `texteocr_provider` absent ou != "mistral", le texte sera **recodé via GPT** (coût API). Solution :
- Ajouter automatiquement `texteocr_provider = "csv"` lors de l'ingestion CSV
- Modifier la logique : `recode_required = provider not in ("mistral", "csv")`

### 4. Fonction `sanitize_metadata_value()` (ligne 242-248)

Gère les types incompatibles JSON/Pinecone :
- Convertit `pd.NA` / `np.nan` → chaîne vide
- Assure types primitifs : str, int, float, bool
- Fallback vers `str(value)`

**Bon point** : Déjà robuste pour gérer des colonnes CSV hétérogènes !

---

## Dépendances du pipeline

### Dépendances Python critiques

```txt
# Chunking & NLP
langchain-text-splitters  # RecursiveCharacterTextSplitter
spacy==3.x + fr_core_news_md

# Embeddings & OCR
openai>=1.x
mistralai
requests

# Manipulation données
pandas
tqdm
python-dotenv

# Bases vectorielles
pinecone>=3.x          # SDK v3+ avec Pinecone class
weaviate-client>=4.x   # Multi-tenancy support
qdrant-client

# Utilitaires
python-dateutil        # Pour Weaviate RFC3339
fitz (PyMuPDF)         # Fallback OCR legacy
```

### Modèles externes

| Service | Modèle par défaut | Usage |
|---------|------------------|-------|
| Mistral OCR | `mistral-ocr-latest` | Extraction PDF → Markdown |
| OpenAI vision | `gpt-4o-mini` | Fallback OCR |
| OpenAI recodage | `gpt-4o-mini` | Nettoyage chunks OCR bruts |
| OpenAI embeddings | `text-embedding-3-large` | Embeddings denses (3072 dim) |
| spaCy | `fr_core_news_md` | Lemmatisation + POS tagging |

---

## Opportunités de refactorisation pour CSV

### Opportunité 1 : Abstraction de la source de `texteocr`

**État actuel** : `texteocr` provient **exclusivement** de l'OCR PDF.

**Cible** : Créer une **Factory d'ingestion** avec :

```python
class BaseIngestionPipeline(ABC):
    @abstractmethod
    def ingest(self) -> List[Dict[str, Any]]:
        """Retourne [{texteocr: str, meta: dict}, ...]"""
        pass

class PDFIngestionPipeline(BaseIngestionPipeline):
    def ingest(self):
        # Code actuel de rad_dataframe.py
        pass

class CSVIngestionPipeline(BaseIngestionPipeline):
    def ingest(self):
        # Lecture CSV → mapping colonne → texteocr
        pass
```

### Opportunité 2 : Métadonnées dynamiques

**État actuel** : Liste de 10 champs hardcodée dans 4 emplacements.

**Cible** : Remplacer par une **injection complète de `meta`** :

```python
# rad_chunk.py - AVANT
chunk_metadata = {
    "title": row_data.get("title", ""),
    "authors": row_data.get("authors", ""),
    # ... 8 autres champs
}

# rad_chunk.py - APRÈS
chunk_metadata = {
    "id": ...,
    "text": ...,
    **row_data.get("meta", {})  # Injection de toutes les métadonnées
}
```

```python
# rad_vectordb.py - AVANT
metadata = {
    "title": chunk.get("title", ""),
    # ... liste hardcodée
}

# rad_vectordb.py - APRÈS
metadata = {k: v for k, v in chunk.items()
            if k not in ("id", "embedding", "sparse_embedding", "text")}
# Ou : metadata = chunk.get("meta", {})
```

### Opportunité 3 : Configuration CSV flexible

```yaml
# config/csv_config.yaml
csv:
  text_column: "text"        # Colonne source de texteocr
  encoding: "auto"            # utf-8, latin-1, auto-detect
  delimiter: ","
  meta_columns: []            # Si vide : toutes sauf text_column
  skip_empty: true            # Ignorer lignes avec texte vide
  add_metadata:
    source_type: "csv"
    ingested_at: "{{timestamp}}"
```

### Opportunité 4 : Classe `Document` unifiée

```python
@dataclass
class Document:
    texteocr: str
    meta: Dict[str, Any]

    def validate(self):
        assert self.texteocr, "texteocr ne peut pas être vide"
        assert isinstance(self.meta, dict), "meta doit être un dict"
```

Tous les pipelines retournent des `List[Document]`, garantissant l'uniformité.

---

## Risques identifiés

| Risque | Probabilité | Mitigation |
|--------|-------------|------------|
| Métadonnées CSV trop volumineuses pour Pinecone | Moyenne | Filtrer/tronquer les valeurs longues + warning |
| Régression sur pipeline PDF | Faible | Suite de tests de non-régression |
| CSV mal encodés | Élevée | Détection auto (chardet) + fallback UTF-8 |
| Colonnes CSV avec noms invalides (espaces, spéciaux) | Moyenne | Sanitisation via `re.sub(r'[^a-zA-Z0-9_]', '_', col)` |
| Recodage GPT activé par erreur sur CSV | Moyenne | Ajout de `texteocr_provider="csv"` automatique |

---

## Prochaines étapes recommandées (Phase 2)

1. **Design du module `csv_ingestion.py`**
   - Fonction `ingest_csv()` → `List[Document]`
   - Configuration du mapping colonnes
   - Validations et logging

2. **Refactorisation de `rad_chunk.py`**
   - Remplacer la création de `chunk_metadata` hardcodée par injection de `meta`
   - Ajouter condition `provider in ("mistral", "csv")` pour skip recodage

3. **Refactorisation de `rad_vectordb.py`**
   - Remplacer les 3 fonctions de préparation par injection dynamique de métadonnées
   - Filtrage optionnel des champs trop volumineux

4. **Tests de bout en bout**
   - Créer un CSV de test avec 10 colonnes variées
   - Vérifier l'injection complète dans Pinecone
   - Tester la recherche avec filtres sur métadonnées CSV

---

**Conclusion** : Le pipeline actuel est bien structuré mais **rigide** au niveau des métadonnées. L'ajout de l'ingestion CSV nécessite :
1. Une abstraction de la source de `texteocr` (Factory pattern)
2. Une refactorisation des métadonnées hardcodées → injection dynamique
3. Un mécanisme de configuration pour mapper les colonnes CSV

La variable `texteocr` est le **point de pivot parfait** : tout le pipeline en aval est déjà agnostique de la source (PDF/OCR ou autre).
