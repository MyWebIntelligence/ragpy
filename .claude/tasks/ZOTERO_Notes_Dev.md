# Plan de Développement : Génération de Fiches de Lecture pour Zotero dans Ragpy

**Version:** 1.1 **Date:** 2025-10-26

## 1\. Objectif de la Feature

Intégrer une fonctionnalité dans Ragpy permettant, lors de l'ingestion d'un export ZIP Zotero (formats JSON standard ou BetterBibTeX), de proposer à l'utilisateur de générer automatiquement une fiche de lecture via un LLM et de l'ajouter comme note enfant à l'item correspondant dans Zotero via l'API v3.

**Cas d'usage ciblé:** Uniquement pour les uploads ZIP contenant un fichier JSON Zotero et des PDF associés.

## 2\. Analyse et Conception Détaillées

Cette section détaille les considérations techniques et fonctionnelles nécessaires à l'implémentation. L'analyse couvre l'identification des sources, le traitement des données Zotero, l'intégration LLM et API Zotero, les ajustements du flux applicatif et de l'interface, ainsi que les aspects de robustesse comme l'idempotence et la gestion des erreurs.

### 2.1. Identification Spécifique des Archives Zotero

L'activation conditionnelle repose sur une identification fiable des archives ZIP Zotero.

- **Modification Substantielle du Backend (`app/main.py::upload_zip`)**: Après extraction du ZIP, validation de la présence d'un fichier JSON. Analyse structurelle minimale : présence des clés `"config"` et `"items"` (ou structures équivalentes pour BetterBibTeX). Vérifications additionnelles (`itemType`, `itemKey`). En cas d'identification positive, la réponse de l'endpoint `/upload_zip` inclura `isZoteroExport: true`. Gestion des JSON multiples (priorité au premier trouvé ou signalement d'ambiguïté).
    
- **Ajustement de l'Interface Utilisateur (`index.html`)**: JavaScript vérifiera `isZoteroExport` dans la réponse `/upload_zip`. Si vrai, affichage dynamique des contrôles spécifiques Zotero (case à cocher "Générer et pousser une fiche...", zone d'aperçu HTML). Si faux, ces éléments restent masqués. Cohérence de l'état UI lors de téléversements successifs.
    

### 2.2. Traitement Structuré des Données JSON Zotero

Extraction précise des identifiants et localisation des fichiers associés.

- **Extraction de l'Identifiant d'Item (`Item Key`) et Bibliothèque**: Fonction dédiée pour parcourir `items` et extraire `itemKey` pour chaque item principal (`itemType` != `"attachment"`/`"note"`). Robustesse face aux variations mineures et `itemKey` manquant (atypique). Extraction également de `library_type` (`users` ou `groups`) et `library_id` (ex: `15681` depuis l'URI `.../users/15681/items/4G3PMW5F`) pour cibler correctement l'API. Stockage fiable de `itemKey`, `library_type`, `library_id`.
    
- **Localisation et Validation des Fiches PDF**: Analyse de `attachments` pour chaque item, examen du champ `path`. Consolidation de la résolution de chemin (`rad_dataframe.py`) pour chemins absolus/relatifs (base = répertoire d'extraction ZIP). Validation systématique `os.path.exists`. Stratégies de recherche alternatives si non trouvé (insensible casse, normalisation Unicode, sous-répertoires `files/`). Maintien de la correspondance `itemKey` ↔ chemin PDF. Gestion gracieuse si PDF absent (désactivation génération pour cet item).
    

### 2.3. Intégration du Modèle Linguistique (LLM) pour la Génération de Contenu Automatisé

Génération déléguée au LLM, nécessitant gestion extraction texte, prompt et appel API.

- **Exploitation de l'Extraction Textuelle Préexistante**: Réutilisation prioritaire du `texteocr` extrait par `rad_dataframe.py` (vérification dans `output.csv` de session). Si absent ou régénération demandée, invocation ciblée de l'extraction OCR pour le PDF associé à l'`itemKey`. Qualité et exhaustivité du `texteocr` déterminantes. Option de configuration pour "ne jamais envoyer le PDF" au LLM pour des raisons de confidentialité, se basant alors uniquement sur les métadonnées Zotero (titre, résumé).
    
- **Conception Stratégique de l'Instruction (Prompt)**: Instruction standardisée et paramétrable. Directives précises :
    
    - **Objectif**: Fiche de lecture.
        
    - **Format**: HTML simplifié (`<p>`, `<strong>`, `<em>`, `<ul><li>`).
        
    - **Idempotence**: Inclure un commentaire HTML unique et stable : `<!-- ragpy-note-id:{UUID} -->` où UUID est généré une fois par demande de fiche.
        
    - **Longueur**: 200-300 mots.
        
    - **Contenu Requis**: Points fondamentaux, méthodologie, conclusions/résultats. Utilisation optionnelle des métadonnées Zotero (titre, auteurs, résumé) pour enrichir le contexte.
        
    - **Ton**: Neutre, informatif, académique.
        
    - **Langue Cible**: Déterminée à partir du champ `language` de l'item Zotero si présent, sinon heuristique (détection sur titre/résumé) ou langue par défaut configurable. _Exemple d'Instruction (simplifié)_: `"[...] Format HTML simplifié incluant '<!-- ragpy-note-id:{UUID} -->'. Mettre en évidence (1) points fondamentaux, (2) méthodologie, (3) conclusions. Ton neutre. Texte : {TEXTE_DU_PDF}"` Définition comme constante (ex: `utils/llm_prompts.py`). Échappement correct du texte injecté.
        
- **Invocation Contrôlée du Modèle Linguistique (LLM)**: Intégration technique de l'appel API LLM (Gemini, OpenAI, via client configuré).
    
    - Sélection dynamique du modèle (config Ragpy).
        
    - Construction requête API (prompt formaté + `texteocr`).
        
    - Exécution asynchrone (non-bloquante UI).
        
    - Gestion robuste des erreurs : timeouts configurables, quota (429 + backoff exponentiel), erreurs serveur (5xx), validation prompt/contenu, exceptions client API. Fallback ou message erreur clair si échec persistant. Extraction et validation réponse LLM (HTML + présence sentinel).
        

### 2.4. Intégration Fonctionnelle de l'API Zotero

Ajout de la note via API Zotero, impliquant configuration, communication sécurisée, idempotence et gestion réponses.

- **Paramétrage Sécurisé**:
    
    - **Configuration Backend**: Ajout `ZOTERO_API_KEY`, `ZOTERO_USER_ID`, `ZOTERO_GROUP_ID` dans `.env`. Chargement via `python-dotenv` au démarrage FastAPI, accessible par `os.getenv()`.
        
    - **Interface de Configuration Utilisateur (`index.html`, `main.py`)**: Section "Settings" étendue : champs pour clé API (type `password`), ID user/groupe. Masquage partiel clé après sauvegarde. Endpoint sécurisé pour mise à jour `.env`.
        
- **Développement d'un Module Client Zotero (Python)**: Module `utils/zotero_client.py` encapsulant interaction API Zotero v3. Fonctions clés :
    
    - `get_library_version(prefix: str, api_key: str) -> str`: Récupère la version actuelle via `GET /{prefix}/items/top?limit=1` et lecture header `Last-Modified-Version`.
        
    - `check_note_exists(prefix: str, item_key: str, sentinel: str, api_key: str) -> bool`: Vérifie via `GET /{prefix}/items/{item_key}/children` si une note contenant le `sentinel` existe déjà.
        
    - `add_note_to_item(prefix: str, item_key: str, note_html_content: str, api_key: str, library_version: str = None) -> dict`: Fonction principale pour `POST /{prefix}/items`.
        
        - Construit URL API (`prefix` = `users/{userID}` ou `groups/{groupID}`).
            
        - Prépare payload JSON : `itemType: "note"`, `parentItem: item_key`, `note: note_html_content` (incluant sentinel), `tags: [{"tag": "RagpyGenerated"}, ...]`.
            
        - Exécute `POST` avec headers requis : `Authorization: Bearer {api_key}`, `Zotero-API-Version: 3`.
            
        - **Contrôle de Concurrence**: Inclut `If-Unmodified-Since-Version: {library_version}` pour garantir l'atomicité basée sur la version. Alternative (ou combiné) : `Zotero-Write-Token: {UUID}` pour idempotence sur retry.
            
        - Analyse réponse : `200 OK`/`201 Created` = succès. Gestion codes erreur (4xx, 5xx). Retourne `{ "success": bool, "message": str, "noteKey": Optional[str], "new_version": Optional[str] }`.
            
        - **Gestion 412 Precondition Failed**: Si `If-Unmodified-Since-Version` utilisé et erreur 412 reçue, tentative de relance : re-fetch `library_version` et re-POST une fois.
            
- **Gestion Avancée des Erreurs de l'API**: Traitement spécifique dans module client et/ou backend :
    
    - `401 Unauthorized`/`403 Forbidden`: Clé API invalide/expirée/permissions insuffisantes.
        
    - `404 Not Found`: `itemKey` parent inexistant.
        
    - `429 Too Many Requests`: Respecter `Retry-After` + backoff exponentiel.
        
    - `400 Bad Request`: Payload JSON invalide (HTML mal formé?).
        
    - `409 Conflict`: Potentiel verrouillage de bibliothèque (rare, suggère retry).
        
    - `412 Precondition Failed`: Version bibliothèque modifiée depuis lecture (géré par retry).
        
    - `5xx Server Error`: Erreurs serveur Zotero. Message erreur frontend informatif pour diagnostic.
        

### 2.5. Modification du Flux Opérationnel Central de Ragpy (`app/main.py`)

Ajout endpoint backend dédié et ajustement logique frontend.

- **Établissement d'un Nouveau Point d'Accès Backend Spécifique**: Endpoint `POST /generate_zotero_note/{session_path}/{itemKey}`.
    
    - **Paramètres d'Entrée**: `session_path`, `itemKey`.
        
    - **Séquence Opérationnelle**:
        
        1. **Récupération Texte Source**: Lecture `texteocr` depuis `output.csv` de session via `itemKey` (ou identifiant dérivé). Erreur si non trouvé (ou déclenchement OCR dynamique optionnel).
            
        2. **Génération Note LLM**: Appel LLM avec `texteocr` et prompt prédéfini (incluant génération UUID pour sentinel). Gestion erreurs LLM.
            
        3. **Récupération Identifiants Zotero**: Lecture sécurisée `ZOTERO_*` depuis config (`os.getenv`). Extraction `library_type`, `library_id` depuis données item si non configurés globalement.
            
        4. **Vérification Préalable & Concurrence**:
            
            - Appel `zotero_client.get_library_version()` pour obtenir version actuelle.
                
            - Appel `zotero_client.check_note_exists()` avec sentinel unique. Si existe déjà, retourner succès sans POST.
                
        5. **Invocation Client Zotero**: Appel `zotero_client.add_note_to_item()` avec `itemKey`, HTML généré, clé API, et `library_version` récupérée.
            
        6. **Logging Détaillé**: Enregistrement dans base de données ou fichier log : `itemKey`, `noteKey` retourné par Zotero (si succès), `ragpy_note_uuid` (le sentinel), `library_version_used`, statut (succès/échec), métadonnées réponse Zotero.
            
        7. **Retour Statut**: Réponse JSON structurée au frontend (`{ "success": bool, "message": str, "noteKey": Optional[str], "zotero_url": Optional[str] }`). `zotero_url` peut être `zotero://select/library/items/{noteKey}`.
            
- **Ajustement de l'Orchestration Frontend (`index.html`)**:
    
    - **Affichage Conditionnel**: Contrôle UI (bouton global "Générer Fiches Zotero" post-`process_dataframe`) visible si `isZoteroExport: true`.
        
    - **Déclenchement**: Clic sur bouton global → JavaScript récupère liste des `itemKey` réussis depuis réponse `process_dataframe` (nécessite adaptation réponse endpoint).
        
    - **Appels Asynchrones**: Boucle sur `itemKey` → `fetch` vers `/generate_zotero_note/{session_path}/{itemKey}`. Utilisation `Promise.allSettled` pour gérer succès/échecs individuels.
        
    - **Feedback Utilisateur Dynamique**: Zone de log/tableau récapitulatif : état par item ("En attente", "Génération LLM...", "Vérif Zotero...", "Ajout Zotero...", "Succès", "Déjà existant", "Échec (détails)"). Barre de progression globale. Affichage snackbar/message final avec résumé (X notes ajoutées, Y échecs). Lien `zotero://` cliquable pour notes ajoutées.
        

### 2.6. Conception de l'Interface et de l'Expérience Utilisateur (UI/UX)

Interface claire, intuitive, informative.

- **Séparation Fonctionnelle Claire**: Option Zotero distincte du flux vectoriel. Section dédiée ou contrôles visuellement séparés. Activation génération note n'interfère pas avec flux embedding/indexation.
    
- **Feedback Processuel Granulaire**: État global ("Génération fiches : 3/10"). État par item (indicateurs : "En attente", "Génération LLM...", "Ajout Zotero...", "Succès", "Échec - API 403"). Messages clairs succès/échec avec cause possible. **Aperçu HTML**: Zone affichant l'HTML généré par le LLM avant (ou après) l'envoi à Zotero, potentiellement éditable par l'utilisateur pour correction rapide.
    
- **Gestion Sécurisée des Identifiants**: Section "Settings" Zotero : champ type `password` pour clé API, affichage masqué après sauvegarde, transmission HTTPS, stockage sécurisé backend (`.env` permissions restreintes ou gestionnaire secrets), pas de stockage client (localStorage) ou code source.
    

## 3\. Étapes d'Implémentation

### Sprint 1 : Backend - Logique Fondamentale & Intégration API Zotero

1. **(Configuration)** Màj `.env.example` & gestion config (`get_credentials`, `save_credentials`) pour `ZOTERO_*`.
    
2. **(Utilitaires)** Créer `utils/zotero_client.py` avec `add_note_to_item()`, `get_library_version()`, `check_note_exists()`. Gestion erreurs API basique + logique retry 412.
    
3. **(Backend)** Implémenter détection Zotero dans `app/main.py::upload_zip`.
    
4. **(Backend)** Développer `/generate_zotero_note` v1 :
    
    - Accepter `session_path`, `itemKey`.
        
    - _Simuler_ extraction texte & appel LLM (placeholders).
        
    - Implémenter logique : get version → check exists → add note.
        
    - Invoquer `zotero_client`. Retourner statut.
        
5. **(Tests)** Tests unitaires `zotero_client.py`. Tests intégration `/generate_zotero_note` (mocks Zotero API), ciblant bibliothèque Zotero de test.
    

### Sprint 2 : Interface Utilisateur & Intégration LLM

1. **(Frontend)** Modifier `index.html` :
    
    - Affichage conditionnel option Zotero post-upload ZIP Zotero.
        
    - Màj "Settings" pour config Zotero.
        
    - Ajouter zone d'aperçu HTML (initialement non éditable).
        
2. **(Frontend)** Logique JavaScript :
    
    - Bouton "Générer Fiches Zotero" post-`process_dataframe`. Récupérer `itemKey` depuis réponse adaptée `process_dataframe`.
        
    - Boucle `fetch` asynchrone vers `/generate_zotero_note`.
        
    - Affichage dynamique feedback utilisateur (statuts par item, log global).
        
3. **(Backend)** Intégrer appel LLM effectif dans `/generate_zotero_note` (via client Ragpy existant). Génération UUID pour sentinel.
    
4. **(Backend)** Implémenter récupération `texteocr` depuis `output.csv`. Adapter réponse `process_dataframe` pour inclure liste `itemKey` traités.
    
5. **(Tests)** Tests E2E simulés (mocks LLM & Zotero API). Tests manuels UI.
    

### Sprint 3 : Finalisation, Tests Approfondis & Documentation

1. **(Backend/Frontend)** Raffiner gestion erreurs (LLM, Zotero API, texte source absent). Améliorer clarté feedback. Rendre l'aperçu HTML éditable (optionnel).
    
2. **(Tests)** Tests intégration plus exhaustifs. Tests E2E sur staging avec appels réels API (LLM, Zotero). Validation scénarios erreur (401, 404, 429, 412, etc.).
    
3. **(Documentation)** Màj `README.md`. Créer doc dédiée config/utilisation Zotero.
    
4. **(Qualité Code)** Revue code. Refactoring (lisibilité, maintenabilité, performance).
    
5. **(Logging)** Implémenter logging backend détaillé (cf. section 2.5, point 6).
    

## 4\. Stratégie de Test

- **Tests Unitaires**: Module `zotero_client.py` (mocks HTTP API Zotero), extraction identifiants, détection Zotero.
    
- **Tests d'Intégration**: Endpoint `/generate_zotero_note` (mocks LLM, Zotero API), validation orchestration interne. Tests spécifiques pour logique retry 412 et check `child_exists`. Tests mock HTTP pour 429, 409.
    
- **Tests End-to-End (E2E)**: Scénarios :
    
    - **Nominal**: ZIP Zotero valide → Option affichée → Génération succès → Notes ajoutées Zotero.
        
    - **Erreurs API Zotero**: Clé invalide (401/403), item non trouvé (404), version modifiée (412 + retry), rate limit (429).
        
    - **Erreur LLM**: Timeout LLM.
        
    - **PDF Manquant**: Génération omise pour item concerné.
        
    - **Archive Non-Zotero**: Option non affichée.
        
    - **Note Déjà Existante**: Vérification `child_exists` → pas de POST, retour "Déjà existant".
        
- **Tests UI**: Affichage conditionnel, réactivité, feedback visuel, sauvegarde/chargement Settings Zotero. Tests multilingues (si pertinent).
    

## 5\. Documentation

- Màj `README.md` : section intégration Zotero, config `.env`, utilisation UI.
    
- Doc technique `zotero_client.py`.
    
- Aide UI ou doc externe : obtenir clé API Zotero + ID user/groupe. Inclure détails headers Zotero API utilisés (`Version`, `If-Unmodified...`).
    

## 6\. Déploiement

- Ajout `requests` à `requirements.txt` si absent.
    
- Communication utilisateur : nouvelle feature, nécessité config Zotero.
    
- Monitoring post-déploiement : fréquence/succès appels API Zotero & LLM.
    
- **Feature Flag**: Introduire variable env `FEATURE_ZOTERO_NOTES=true/false` pour activation/désactivation facile.
    

## 7\. Perspectives d'Évolution

- UI pour personnalisation prompt LLM.
    
- Option "Mettre à jour note Ragpy existante" (via sentinel + `PUT/PATCH` Zotero API).
    
- Gestion configurable bibliothèques groupe Zotero (`groupID`).
    
- Utilisation métadonnées Zotero (tags, collections) pour enrichir contexte LLM.
    
- **Traçabilité**: Ajouter tag Zotero `ragpy:<version>` aux notes créées.
    
- **Authentification**: Explorer OAuth 1.0a pour délégation création clés API.

# Exemples de code

Voici un module **FastAPI + CLI** prêt à l’emploi pour le mode **ZOTERO** de Ragpy.
Extraction `library_type/userID/itemKey` depuis `uri` est conforme à votre JSON d’exemple, p.ex. `http://zotero.org/users/15681/items/4G3PMW5F`. 
Points d’API Zotero utilisés : préfixes `/users/<userID>` ou `/groups/<groupID>` et auth clé API, création d’une **note enfant** via `parentItem`, contrôle de concurrence par `If-Unmodified-Since-Version` ou `Zotero-Write-Token`, lecture de `Last-Modified-Version`, vérification de clé via `/keys/current`.

---

### `pyproject.toml`

```toml
[project]
name = "ragpy-zotero"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
  "fastapi>=0.112",
  "uvicorn[standard]>=0.30",
  "pydantic>=2.7",
  "python-multipart>=0.0.9",
  "requests>=2.32",
  "tenacity>=9.0",
]

[project.scripts]
ragpy-zotero = "ragpy_zotero.cli:main"
```

---

### Arborescence

```
src/
  ragpy_zotero/
    __init__.py
    config.py
    parser.py
    llm_note.py
    zotero_client.py
    pipeline.py
    app.py
```

---

### `src/ragpy_zotero/config.py`

```python
from __future__ import annotations
import os
from pydantic import BaseModel, Field, ValidationError

class Settings(BaseModel):
    # Zotero
    zotero_api_key: str = Field(validation_alias="ZOTERO_API_KEY")
    default_library_type: str | None = Field(default=None, validation_alias="ZOTERO_LIBRARY_TYPE")  # "users"|"groups"
    default_library_id: str | None = Field(default=None, validation_alias="ZOTERO_LIBRARY_ID")
    zotero_base_url: str = Field(default="https://api.zotero.org", validation_alias="ZOTERO_BASE_URL")

    # LLM (facultatif: fallback template si non configuré)
    llm_backend: str = Field(default="template", validation_alias="LLM_BACKEND")  # "template" | "openai"
    openai_api_key: str | None = Field(default=None, validation_alias="OPENAI_API_KEY")
    openai_model: str | None = Field(default=None, validation_alias="OPENAI_MODEL")

    # Pipeline
    feature_push_note: bool = Field(default=True, validation_alias="FEATURE_ZOTERO_NOTES")
    feature_vector_insert: bool = Field(default=True, validation_alias="FEATURE_VECTOR_INSERT")

def get_settings() -> Settings:
    try:
        return Settings(**os.environ)
    except ValidationError as e:
        # Autoriser le run CLI sans clé si push note désactivé
        env = dict(os.environ)
        if not env.get("FEATURE_ZOTERO_NOTES", "true").lower() in {"1","true","yes"}:
            env["ZOTERO_API_KEY"] = env.get("ZOTERO_API_KEY", "DUMMY")
            return Settings(**env)
        raise
```

---

### `src/ragpy_zotero/parser.py`

```python
from __future__ import annotations
import io, json, re, zipfile
from dataclasses import dataclass
from typing import Any

URI_RE = re.compile(r"zotero\.org/(users|groups)/(\d+)/items/([A-Z0-9]{8})", re.I)

@dataclass
class ZoteroItemMini:
    title: str | None
    abstract: str | None
    language: str | None
    creators: list[dict]
    date: str | None
    url: str | None
    doi: str | None
    tags: list[str]
    itemKey: str
    library_type: str  # users | groups
    library_id: str

def _coerce_item(raw: dict) -> ZoteroItemMini:
    # 1) itemKey direct
    item_key = raw.get("itemKey")
    lib_type = None
    lib_id = None
    # 2) si uri présent on dérive (conforme à l'export BetterBibTeX)  :contentReference[oaicite:2]{index=2}
    uri = raw.get("uri") or ""
    m = URI_RE.search(uri)
    if m:
        lib_type, lib_id, uri_key = m.group(1).lower(), m.group(2), m.group(3)
        item_key = item_key or uri_key
    if not item_key or not lib_type or not lib_id:
        raise ValueError(f"Impossible d'extraire library/item depuis l'item: {raw.get('title')!r}")

    tags = []
    for t in raw.get("tags", []):
        if isinstance(t, dict) and "tag" in t:
            tags.append(t["tag"])
        elif isinstance(t, str):
            tags.append(t)

    doi = None
    extra = raw.get("extra") or ""
    if isinstance(extra, str):
        for line in extra.splitlines():
            if "DOI:" in line:
                doi = line.split("DOI:", 1)[1].strip()

    return ZoteroItemMini(
        title=raw.get("title"),
        abstract=raw.get("abstractNote"),
        language=raw.get("language"),
        creators=raw.get("creators", []),
        date=raw.get("date"),
        url=raw.get("url"),
        doi=doi,
        tags=tags,
        itemKey=item_key,
        library_type=lib_type,
        library_id=lib_id,
    )

def extract_items_from_zip(zip_bytes: bytes) -> list[ZoteroItemMini]:
    """Détecte un JSON Zotero dans un ZIP et retourne la liste minimale d'items."""
    with zipfile.ZipFile(io.BytesIO(zip_bytes)) as zf:
        json_members = [n for n in zf.namelist() if n.lower().endswith(".json")]
        if not json_members:
            raise ValueError("Aucun JSON Zotero trouvé dans le ZIP.")
        # Heuristique: prendre le plus gros JSON
        name = max(json_members, key=lambda n: zf.getinfo(n).file_size)
        data = json.loads(zf.read(name).decode("utf-8"))
        if "items" not in data or not isinstance(data["items"], list):
            raise ValueError("JSON invalide: clé 'items' manquante.")
        items = []
        for raw in data["items"]:
            try:
                items.append(_coerce_item(raw))
            except Exception:
                continue
        if not items:
            raise ValueError("Aucun item exploitable.")
        return items
```

---

### `src/ragpy_zotero/llm_note.py`

```python
from __future__ import annotations
import html, os, uuid
from typing import Optional
from .config import get_settings
from dataclasses import asdict

SENTINEL_PREFIX = "ragpy-note-id:"

def _fallback_template(meta: dict, lang: str) -> str:
    """Génère une fiche simple si aucun LLM n'est configuré."""
    title = html.escape(meta.get("title") or "Sans titre")
    authors = ", ".join(
        [f"{c.get('lastName','')}".strip() for c in meta.get("creators", []) if isinstance(c, dict)]
    ) or "Auteur·s N/A"
    date = html.escape((meta.get("date") or "")[:10])
    link = html.escape(meta.get("url") or meta.get("doi") or "")
    abstract = html.escape((meta.get("abstract") or "")[:1200])
    return f"""<p><em>Fiche générée automatiquement (template).</em></p>
<h3>Fiche de lecture</h3>
<p><strong>Réf.</strong> {title} — {authors} — {date} — {link}</p>
<ul>
  <li><strong>Problématique</strong> : à compléter</li>
  <li><strong>Méthode</strong> : à compléter</li>
  <li><strong>Résultats clés</strong> : à compléter</li>
  <li><strong>Limites</strong> : à compléter</li>
</ul>
<p><strong>Résumé</strong> : {abstract}</p>
"""

def _openai_note(meta: dict, lang: str) -> str:
    """Optionnel: LLM OpenAI si variables présentes."""
    try:
        from openai import OpenAI  # pip install openai
    except Exception:
        return _fallback_template(meta, lang)
    s = get_settings()
    if not (s.openai_api_key and s.openai_model):
        return _fallback_template(meta, lang)
    client = OpenAI(api_key=s.openai_api_key)
    sys = ("Tu es un assistant qui rédige des fiches de lecture courtes en HTML propre, 200–300 mots, "
           "langue cible={lang}. Pas de citations longues.").format(lang=lang or "fr")
    user = f"""Métadonnées:
title={meta.get('title')}
date={meta.get('date')}
language={meta.get('language')}
authors={[c.get('lastName') for c in meta.get('creators', []) if isinstance(c, dict)]}
doi={meta.get('doi')}
url={meta.get('url')}
abstract={meta.get('abstract')}
Contraintes:
- Produit uniquement du HTML à insérer dans une note Zotero.
- Structure: Réf., liste Problématique/Méthode/Résultats/Limitations, court paragraphe.
"""
    resp = client.chat.completions.create(
        model=s.openai_model,
        messages=[{"role":"system","content":sys}, {"role":"user","content":user}],
        temperature=0.2,
    )
    return resp.choices[0].message.content.strip()

def build_note_html(meta: dict) -> tuple[str, str]:
    """Retourne (sentinel, note_html_including_sentinel_comment)."""
    s = get_settings()
    target_lang = meta.get("language") or "fr"
    if s.llm_backend == "openai":
        body = _openai_note(meta, target_lang)
    else:
        body = _fallback_template(meta, target_lang)
    sentinel = f"{SENTINEL_PREFIX}{uuid.uuid4()}"
    # commentaire HTML en tête pour idempotence
    note_html = f"<!-- {sentinel} -->\n{body}"
    return sentinel, note_html

def sentinel_in_html(html_text: str) -> bool:
    return SENTINEL_PREFIX in (html_text or "")
```

---

### `src/ragpy_zotero/zotero_client.py`

```python
from __future__ import annotations
import time, requests
from typing import Any
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception
from .config import get_settings

# API v3 basics: /users/<id> or /groups/<id>, auth, etc.  
# Child notes via parentItem on POST /<prefix>/items.  
# Concurrency: If-Unmodified-Since-Version or Zotero-Write-Token; 429/409/412.  

class ZoteroHTTPError(Exception):
    def __init__(self, response: requests.Response):
        self.response = response
        super().__init__(f"HTTP {response.status_code}: {response.text[:200]}")

def _maybe_sleep_retry_after(resp: requests.Response):
    if resp.status_code in (429, 409):
        ra = resp.headers.get("Retry-After")
        if ra:
            try:
                time.sleep(int(ra))
            except Exception:
                time.sleep(2)

def _retry_predicate(e: Exception) -> bool:
    if isinstance(e, ZoteroHTTPError):
        sc = e.response.status_code
        return sc in (409, 412, 429, 500, 502, 503, 504)
    return False

class ZoteroClient:
    def __init__(self, library_type: str, library_id: str, api_key: str | None = None):
        s = get_settings()
        self.base = s.zotero_base_url.rstrip("/")
        self.library_type = library_type  # "users"|"groups"
        self.library_id = library_id
        self.api_key = api_key or s.zotero_api_key
        self.session = requests.Session()
        self.session.headers.update({
            "Zotero-API-Key": self.api_key,
            "Zotero-API-Version": "3",
            "Content-Type": "application/json",
            "Accept": "application/json",
        })

    @property
    def prefix(self) -> str:
        return f"{self.base}/{self.library_type}/{self.library_id}"

    def verify_key(self) -> dict:
        # /keys/current permet de valider la portée de la clé.  
        r = self.session.get(f"{self.base}/keys/current", timeout=20)
        if r.status_code >= 400:
            raise ZoteroHTTPError(r)
        return r.json()

    def get_library_version(self) -> str:
        # Lire Last-Modified-Version via items/top?limit=1.  
        r = self.session.get(f"{self.prefix}/items/top", params={"limit": 1}, timeout=20)
        if r.status_code >= 400:
            raise ZoteroHTTPError(r)
        return r.headers.get("Last-Modified-Version", "0")

    def get_item(self, item_key: str) -> dict:
        r = self.session.get(f"{self.prefix}/items/{item_key}", timeout=20)
        if r.status_code >= 400:
            raise ZoteroHTTPError(r)
        return r.json()

    def list_children_notes(self, item_key: str) -> list[dict]:
        r = self.session.get(f"{self.prefix}/items/{item_key}/children",
                             params={"itemType": "note"}, timeout=30)
        if r.status_code >= 400:
            raise ZoteroHTTPError(r)
        return r.json()

    @retry(stop=stop_after_attempt(3),
           wait=wait_exponential(multiplier=1, min=1, max=8),
           retry=retry_if_exception(_retry_predicate))
    def create_child_note(self, item_key: str, note_html: str,
                          tags: list[str] | None = None) -> dict:
        body = [{
            "itemType": "note",
            "note": note_html,               # Notes acceptent HTML.  (observé et supporté côté API)
            "parentItem": item_key,          # note enfant
            "tags": [{"tag": t} for t in (tags or [])],
        }]
        headers = {}
        # Choix: versioning pour éviter doublons; à défaut write-token.  
        try:
            lib_ver = self.get_library_version()
            headers["If-Unmodified-Since-Version"] = lib_ver
        except ZoteroHTTPError:
            # De secours: write token unique
            import uuid
            headers["Zotero-Write-Token"] = uuid.uuid4().hex

        r = self.session.post(f"{self.prefix}/items", json=body, headers=headers, timeout=30)
        if r.status_code in (409, 429):
            _maybe_sleep_retry_after(r)
            raise ZoteroHTTPError(r)
        if r.status_code == 412:
            # Version obsolète: rafraîchir et retenter
            lib_ver = self.get_library_version()
            r = self.session.post(f"{self.prefix}/items", json=body,
                                  headers={"If-Unmodified-Since-Version": lib_ver}, timeout=30)

        if r.status_code >= 400:
            raise ZoteroHTTPError(r)
        return r.json()
```

---

### `src/ragpy_zotero/pipeline.py`

```python
from __future__ import annotations
from typing import Iterable
from .config import get_settings
from .parser import extract_items_from_zip, ZoteroItemMini
from .zotero_client import ZoteroClient
from .llm_note import build_note_html, sentinel_in_html

def _index_to_vector_store(items: Iterable[ZoteroItemMini]) -> dict:
    """Stub d'indexation: à câbler avec votre base vectorielle."""
    # Implémenter: concat titre + abstract + notes existantes (si vous les chargez) etc.
    return {"indexed": len(list(items))}

def _note_already_exists(client: ZoteroClient, item_key: str) -> bool:
    notes = client.list_children_notes(item_key)
    return any(sentinel_in_html(n.get("data", {}).get("note", "")) for n in notes)

def process_zip(zip_bytes: bytes, generate_note: bool = True, insert_vector: bool = True) -> list[dict]:
    s = get_settings()
    out: list[dict] = []
    items = extract_items_from_zip(zip_bytes)

    # Optionnel: un seul appel verify_key au début
    if generate_note and s.feature_push_note:
        # Vérifie que la clé est valable (portée etc.).  
        try:
            # On tentera pour la bibliothèque du premier item
            z0 = ZoteroClient(items[0].library_type, items[0].library_id)
            z0.verify_key()
        except Exception as e:
            raise RuntimeError(f"Clé API Zotero invalide: {e}") from e

    if insert_vector and s.feature_vector_insert:
        _index_to_vector_store(items)

    for it in items:
        res = {"itemKey": it.itemKey, "library": f"{it.library_type}/{it.library_id}"}
        if generate_note and s.feature_push_note:
            zc = ZoteroClient(it.library_type, it.library_id)
            if _note_already_exists(zc, it.itemKey):
                res["note"] = "exists"
            else:
                sentinel, html = build_note_html({
                    "title": it.title, "abstract": it.abstract, "language": it.language,
                    "creators": it.creators, "date": it.date, "url": it.url, "doi": it.doi
                })
                resp = zc.create_child_note(it.itemKey, html, tags=["ragpy", "fiche-lecture"])
                res["note"] = "created"
                res["sentinel"] = sentinel
                res["api_response"] = resp
        out.append(res)
    return out
```

---

### `src/ragpy_zotero/app.py`

```python
from __future__ import annotations
from fastapi import FastAPI, UploadFile, File, Form
from fastapi.responses import JSONResponse
from .pipeline import process_zip

app = FastAPI(title="Ragpy Zotero Feature")

@app.post("/ingest/zotero")
async def ingest(
    file: UploadFile = File(...),
    push_note: bool = Form(True),
    index_vector: bool = Form(True),
):
    content = await file.read()
    try:
        result = process_zip(content, generate_note=push_note, insert_vector=index_vector)
        return JSONResponse({"status": "ok", "result": result})
    except Exception as e:
        return JSONResponse({"status": "error", "detail": str(e)}, status_code=400)
```

---

### `src/ragpy_zotero/cli.py`

```python
from __future__ import annotations
import argparse, sys
from pathlib import Path
from .pipeline import process_zip

def main(argv: list[str] | None = None) -> int:
    p = argparse.ArgumentParser(description="Ragpy Zotero mode")
    p.add_argument("--zip", required=True, help="Chemin du ZIP Zotero")
    p.add_argument("--no-push", action="store_true", help="Ne pas pousser la note dans Zotero")
    p.add_argument("--no-vector", action="store_true", help="Ne pas indexer dans la base vectorielle")
    args = p.parse_args(argv)

    data = Path(args.zip).read_bytes()
    res = process_zip(data, generate_note=not args.no_push, insert_vector=not args.no_vector)
    print(res)
    return 0

if __name__ == "__main__":
    raise SystemExit(main())
```

---

### Utilisation

* API :

```bash
export ZOTERO_API_KEY=...  # requis si push note actif
uvicorn ragpy_zotero.app:app --reload
# POST via curl
curl -F file=@/chemin/export_zotero.zip -F push_note=true -F index_vector=true http://127.0.0.1:8000/ingest/zotero
```

* CLI :

```bash
export ZOTERO_API_KEY=...
ragpy-zotero --zip /chemin/export_zotero.zip
```

**Notes API Zotero** :

* Préfixes `/users/<userID>` ou `/groups/<groupID>`, auth clé API, et bases des requêtes.
* Création note enfant par `POST /<prefix>/items` avec `itemType:"note"` et `parentItem:"<itemKey>"`.
* Concurrence et idempotence : `If-Unmodified-Since-Version` ou `Zotero-Write-Token`; erreurs `409/412/429`.
* Validation de la clé par `/keys/current`.

Besoin d’un gabarit de prompt LLM spécifique ou d’un branchement OpenAI/ollama ? Donne la cible, je fournis la variante.
