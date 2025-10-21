# Plan de développement : Migration OpenRouter pour le chunking

## 📋 Périmètre

**Objectif** : Remplacer OpenAI par OpenRouter (Gemini 2.5 Flash) pour le recodage de texte lors du chunking.

**Ce qui change** :
- ✅ Recodage de texte : `gpt-4o-mini` → `openai/gemini-2.5-flash` via OpenRouter
- ❌ Embeddings : Garder OpenAI `text-embedding-3-large` (inchangé)
- ❌ OCR : Garder Mistral (priorité) / OpenAI (fallback) (inchangé)

**Comportement attendu** :
1. L'UI affiche un champ texte pré-rempli avec `openai/gemini-2.5-flash` dans la section 3.1
2. L'utilisateur peut changer le modèle (champ texte libre)
3. Le système essaie OpenRouter en premier
4. Si OpenRouter échoue (quelque raison que ce soit) → afficher modal de confirmation
5. Si l'utilisateur confirme → utiliser OpenAI `gpt-4o-mini` en fallback
6. Si l'utilisateur annule → arrêter le processus

---

## 🔧 Modifications techniques détaillées

### 1. Configuration & Variables d'environnement

#### Fichier : `.env` (racine du projet)

Ajouter les variables suivantes :

```bash
# OpenRouter Configuration
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxx
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1
OPENROUTER_DEFAULT_MODEL=openai/gemini-2.5-flash
```

**Notes** :
- `OPENROUTER_API_KEY` : Obligatoire pour utiliser OpenRouter
- `OPENROUTER_BASE_URL` : URL de base de l'API OpenRouter
- `OPENROUTER_DEFAULT_MODEL` : Modèle par défaut (format OpenRouter : `provider/model-name`)

---

### 2. Backend : scripts/rad_chunk.py

#### 2.1 Ajout du client OpenRouter (après ligne 70)

**Localisation** : Après `client = OpenAI(api_key=OPENAI_API_KEY)`

```python
# ----------------------------------------------------------------------
# OpenRouter Client Setup (for chunking with Gemini 2.5 Flash)
# ----------------------------------------------------------------------
OPENROUTER_API_KEY = os.getenv("OPENROUTER_API_KEY")
OPENROUTER_BASE_URL = os.getenv("OPENROUTER_BASE_URL", "https://openrouter.ai/api/v1")
OPENROUTER_DEFAULT_MODEL = os.getenv("OPENROUTER_DEFAULT_MODEL", "openai/gemini-2.5-flash")

client_openrouter = None
if OPENROUTER_API_KEY:
    client_openrouter = OpenAI(
        api_key=OPENROUTER_API_KEY,
        base_url=OPENROUTER_BASE_URL
    )
    print(f"✓ Client OpenRouter initialisé (modèle par défaut : {OPENROUTER_DEFAULT_MODEL})")
else:
    print("⚠ OPENROUTER_API_KEY non trouvée, fallback direct sur OpenAI")
```

---

#### 2.2 Nouvelle fonction `gpt_recode_batch_with_openrouter()`

**Localisation** : Insérer juste AVANT la fonction `gpt_recode_batch()` existante (ligne 109)

```python
def gpt_recode_batch_with_openrouter(
    texts,
    instructions,
    model=None,
    temperature=0.3,
    max_tokens=8000
):
    """
    Recode un lot de textes via OpenRouter.
    Retourne (success: bool, cleaned_texts: list, error: str|None)
    """
    if not client_openrouter:
        return (False, None, "Client OpenRouter non initialisé (clé API manquante)")

    if model is None:
        model = OPENROUTER_DEFAULT_MODEL

    cleaned_texts = []
    failed_indices = []

    print(f"  → Tentative de recodage via OpenRouter ({model})...")

    try:
        # Traitement en parallèle avec ThreadPoolExecutor
        with ThreadPoolExecutor(max_workers=DEFAULT_MAX_WORKERS) as executor:
            def call_openrouter(text_item):
                text, idx = text_item
                try:
                    response = client_openrouter.chat.completions.create(
                        model=model,
                        messages=[
                            {"role": "system", "content": instructions},
                            {"role": "user", "content": text}
                        ],
                        temperature=temperature,
                        max_tokens=max_tokens,
                        extra_headers={
                            "HTTP-Referer": "https://ragpy.local",  # Requis par OpenRouter
                            "X-Title": "RAGpy Chunking Pipeline"     # Optionnel mais recommandé
                        }
                    )
                    return (idx, response.choices[0].message.content.strip())
                except Exception as e:
                    print(f"    Erreur OpenRouter pour chunk {idx}: {e}")
                    return (idx, None)

            # Appels parallèles
            results = list(executor.map(call_openrouter, enumerate(texts)))

        # Collecte des résultats
        for idx, cleaned in results:
            if cleaned is None:
                failed_indices.append(idx)
                cleaned_texts.append(texts[idx])  # Texte original si échec
            else:
                cleaned_texts.append(cleaned)

        # Retry séquentiel pour les échecs
        if failed_indices:
            print(f"  → Retry séquentiel pour {len(failed_indices)} chunks échoués...")
            for idx in failed_indices:
                try:
                    response = client_openrouter.chat.completions.create(
                        model=model,
                        messages=[
                            {"role": "system", "content": instructions},
                            {"role": "user", "content": texts[idx]}
                        ],
                        temperature=temperature,
                        max_tokens=max_tokens,
                        extra_headers={
                            "HTTP-Referer": "https://ragpy.local",
                            "X-Title": "RAGpy Chunking Pipeline"
                        }
                    )
                    cleaned_texts[idx] = response.choices[0].message.content.strip()
                except Exception as e_retry:
                    print(f"    Échec définitif pour chunk {idx}: {e_retry}")
                    # Garde le texte original

        print(f"  ✓ Recodage OpenRouter réussi ({len(cleaned_texts)} chunks)")
        return (True, cleaned_texts, None)

    except Exception as e:
        error_msg = str(e)
        print(f"  ✗ Erreur globale OpenRouter: {error_msg}")
        return (False, None, error_msg)
```

---

#### 2.3 Ajout d'un argument `--model` au parser CLI

**Localisation** : Modifier la fonction `main()` (rechercher `argparse.ArgumentParser`)

```python
# Dans la section argparse (généralement vers la fin du fichier)
parser.add_argument(
    '--model',
    type=str,
    default=None,
    help='Modèle pour le recodage (format OpenRouter: provider/model-name). '
         'Par défaut: utilise OPENROUTER_DEFAULT_MODEL si disponible, sinon gpt-4o-mini'
)
```

---

#### 2.4 Modifier `process_document_chunks()` pour utiliser OpenRouter

**Localisation** : Ligne 223, remplacer l'appel à `gpt_recode_batch()`

**Avant** :
```python
cleaned_batch = gpt_recode_batch(
    batch_to_recode,
    instructions="ce chunk est issu d'un ocr brut...",
    model="gpt-4o-mini",
    temperature=0.3,
    max_tokens=8000
)
```

**Après** :
```python
# Récupérer le modèle depuis les arguments CLI
model_to_use = getattr(process_document_chunks, '_model_override', None)

# Essayer OpenRouter en premier si disponible
if client_openrouter and model_to_use:
    success, cleaned_batch, error = gpt_recode_batch_with_openrouter(
        batch_to_recode,
        instructions="ce chunk est issu d'un ocr brut qui laisse beaucoup de blocs de texte inutiles comme des titres de pages, des numeros, etc. Nettoie ce chunk pour en faire un texte propre qui commence par une phrase complète et se termine par un point. Supprime le bruit d'OCR et les imperfections en conservant le sens original. Ne echange ni ajoute aucun mot du texte d'origine. C'est une correction et un nettoyage de texte (suppression des erreurs) pas une réécriture",
        model=model_to_use,
        temperature=0.3,
        max_tokens=8000
    )

    if not success:
        # Signal d'échec OpenRouter → nécessite fallback
        raise Exception(f"OPENROUTER_UNAVAILABLE: {error}")
else:
    # Fallback OpenAI (comportement original)
    print("  → Utilisation d'OpenAI (gpt-4o-mini) pour le recodage...")
    cleaned_batch = gpt_recode_batch(
        batch_to_recode,
        instructions="ce chunk est issu d'un ocr brut qui laisse beaucoup de blocs de texte inutiles comme des titres de pages, des numeros, etc. Nettoie ce chunk pour en faire un texte propre qui commence par une phrase complète et se termine par un point. Supprime le bruit d'OCR et les imperfections en conservant le sens original. Ne echange ni ajoute aucun mot du texte d'origine. C'est une correction et un nettoyage de texte (suppression des erreurs) pas une réécriture",
        model="gpt-4o-mini",
        temperature=0.3,
        max_tokens=8000
    )
```

**Note** : Ajouter au début de la fonction `process_document_chunks()` un moyen de passer le modèle :

```python
def process_document_chunks(csv_file, json_file, model=None):
    # Stocker le modèle pour usage dans le traitement
    if model:
        process_document_chunks._model_override = model
    # ... reste du code
```

---

#### 2.5 Modifier la fonction `main()` pour passer le modèle

**Localisation** : Vers la fin du fichier, dans `if __name__ == "__main__":`

```python
# Dans la phase 'initial'
if args.phase in ["initial", "all"]:
    # ...
    process_document_chunks(
        args.input,
        json_file,
        model=args.model  # ← Nouveau paramètre
    )
```

---

### 3. Backend : app/main.py (FastAPI)

#### 3.1 Nouveau endpoint `/check_openrouter`

**Localisation** : Insérer après l'endpoint `/save_credentials` (après ligne 526)

```python
@app.post("/check_openrouter")
async def check_openrouter():
    """
    Vérifie si OpenRouter est accessible et fonctionnel.
    Retourne {available: bool, error: str|None}
    """
    try:
        openrouter_key = os.getenv("OPENROUTER_API_KEY")
        if not openrouter_key:
            return {"available": False, "error": "OPENROUTER_API_KEY non configurée"}

        # Test de connexion simple (appel minimal)
        import requests
        response = requests.get(
            "https://openrouter.ai/api/v1/models",
            headers={"Authorization": f"Bearer {openrouter_key}"},
            timeout=5
        )

        if response.status_code == 200:
            return {"available": True, "error": None}
        else:
            return {
                "available": False,
                "error": f"Erreur HTTP {response.status_code}: {response.text[:200]}"
            }

    except Exception as e:
        return {"available": False, "error": str(e)}
```

---

#### 3.2 Modifier `/initial_text_chunking` pour supporter le modèle

**Localisation** : Ligne 582-624

**Modifications** :

1. **Ajouter un paramètre `model`** dans la fonction :

```python
@app.post("/initial_text_chunking")
async def initial_text_chunking(model: str = Form(default="openai/gemini-2.5-flash")):
    """
    Génère les chunks initiaux avec recodage GPT.
    Paramètres:
        - model: Modèle OpenRouter à utiliser (format: provider/model-name)
    """
```

2. **Passer le modèle au script** :

```python
# Ligne ~600, modifier la commande subprocess
script_args = [
    sys.executable,
    str(script_path),
    "--input", str(csv_path),
    "--output", str(upload_folder),
    "--phase", "initial",
    "--model", model  # ← Nouveau argument
]
```

3. **Gérer l'exception OPENROUTER_UNAVAILABLE** :

```python
# Dans le bloc try/except du subprocess (ligne ~605)
try:
    result = subprocess.run(
        script_args,
        cwd=project_root,
        capture_output=True,
        text=True,
        timeout=3600
    )

    # Vérifier si l'erreur est liée à OpenRouter
    if result.returncode != 0:
        error_output = result.stderr
        if "OPENROUTER_UNAVAILABLE" in error_output:
            # Extraire le message d'erreur
            error_msg = error_output.split("OPENROUTER_UNAVAILABLE:", 1)[1].strip() if ":" in error_output else "Service inaccessible"
            return {
                "fallback_needed": True,
                "error": error_msg,
                "full_error": error_output
            }
        else:
            # Autre erreur
            raise HTTPException(status_code=500, detail=error_output)

    # Succès normal
    return {"success": True, "message": "Chunking initial réussi avec OpenRouter"}

except subprocess.TimeoutExpired:
    raise HTTPException(status_code=500, detail="Timeout lors du chunking (>1h)")
except Exception as e:
    raise HTTPException(status_code=500, detail=str(e))
```

---

#### 3.3 Nouvel endpoint `/initial_text_chunking_fallback`

**Localisation** : Après `/initial_text_chunking` (après ligne 624)

```python
@app.post("/initial_text_chunking_fallback")
async def initial_text_chunking_fallback():
    """
    Re-exécute le chunking initial en forçant l'utilisation d'OpenAI.
    Appelé après confirmation de l'utilisateur en cas d'échec OpenRouter.
    """
    try:
        # Chemins identiques à /initial_text_chunking
        upload_folder = PROJECT_ROOT / "uploads"
        csv_path = upload_folder / "output.csv"

        if not csv_path.exists():
            raise HTTPException(status_code=404, detail="output.csv introuvable")

        # Chemin du script
        script_path = PROJECT_ROOT / "scripts" / "rad_chunk.py"
        if not script_path.exists():
            raise HTTPException(status_code=500, detail=f"Script introuvable: {script_path}")

        # Commande SANS --model (force OpenAI par défaut)
        script_args = [
            sys.executable,
            str(script_path),
            "--input", str(csv_path),
            "--output", str(upload_folder),
            "--phase", "initial"
            # ← PAS de --model, donc rad_chunk.py utilisera OpenAI
        ]

        print(f"[FALLBACK] Exécution du chunking avec OpenAI : {' '.join(script_args)}")

        result = subprocess.run(
            script_args,
            cwd=PROJECT_ROOT,
            capture_output=True,
            text=True,
            timeout=3600
        )

        if result.returncode != 0:
            raise HTTPException(status_code=500, detail=result.stderr)

        return {
            "success": True,
            "message": "Chunking réussi avec OpenAI (fallback)",
            "output": result.stdout
        }

    except subprocess.TimeoutExpired:
        raise HTTPException(status_code=500, detail="Timeout lors du chunking fallback (>1h)")
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

---

#### 3.4 Modifier `/get_credentials` pour inclure OpenRouter

**Localisation** : Ligne 412-462

**Modification** : Ajouter la clé OpenRouter dans la réponse

```python
# Ligne ~453, ajouter dans le return
return {
    "OPENAI_API_KEY": openai_key,
    "PINECONE_API_KEY": pinecone_key,
    "MISTRAL_API_KEY": mistral_key,
    "WEAVIATE_URL": weaviate_url,
    "WEAVIATE_API_KEY": weaviate_key,
    "QDRANT_URL": qdrant_url,
    "QDRANT_API_KEY": qdrant_key,
    "OPENROUTER_API_KEY": os.getenv("OPENROUTER_API_KEY", ""),  # ← Nouveau
    "OPENROUTER_DEFAULT_MODEL": os.getenv("OPENROUTER_DEFAULT_MODEL", "openai/gemini-2.5-flash")  # ← Nouveau
}
```

---

#### 3.5 Modifier `/save_credentials` pour sauvegarder OpenRouter

**Localisation** : Ligne 464-526

**Modifications** :

1. **Accepter les nouveaux paramètres** :

```python
@app.post("/save_credentials")
async def save_credentials(
    OPENAI_API_KEY: str = Form(default=""),
    PINECONE_API_KEY: str = Form(default=""),
    MISTRAL_API_KEY: str = Form(default=""),
    WEAVIATE_URL: str = Form(default=""),
    WEAVIATE_API_KEY: str = Form(default=""),
    QDRANT_URL: str = Form(default=""),
    QDRANT_API_KEY: str = Form(default=""),
    OPENROUTER_API_KEY: str = Form(default=""),  # ← Nouveau
    OPENROUTER_DEFAULT_MODEL: str = Form(default="openai/gemini-2.5-flash")  # ← Nouveau
):
```

2. **Ajouter dans la liste des clés à sauvegarder** :

```python
# Ligne ~500
credential_keys_to_save = [
    "OPENAI_API_KEY",
    "PINECONE_API_KEY",
    "MISTRAL_API_KEY",
    "WEAVIATE_URL",
    "WEAVIATE_API_KEY",
    "QDRANT_URL",
    "QDRANT_API_KEY",
    "OPENROUTER_API_KEY",  # ← Nouveau
    "OPENROUTER_DEFAULT_MODEL"  # ← Nouveau
]
```

---

### 4. Frontend : app/templates/index.html

#### 4.1 Ajouter le champ modèle dans la section 3.1

**Localisation** : Ligne 149-158, après le `<p>` d'avertissement, avant le formulaire d'upload

```html
<!-- Step 3.1: Initial Text Chunking -->
<section id="initial-chunk-section" style="display:none;">
  <h2>3.1 Initial Text Chunking</h2>
  <p style="color: black; font-size: 0.9em;">Warning: This process can take a long time depending on your file. Do not close your browser. We recommend processing a maximum of 10 articles or one book at a time.</p>

  <!-- ↓↓↓ NOUVEAU CHAMP MODÈLE ↓↓↓ -->
  <div style="margin-bottom: 15px;">
    <label for="chunkingModel" style="display: block; margin-bottom: 5px; font-weight: bold;">
      Modèle de recodage :
    </label>
    <input
      type="text"
      id="chunkingModel"
      name="model"
      value="openai/gemini-2.5-flash"
      placeholder="openai/gemini-2.5-flash"
      style="width: 100%; padding: 8px; border: 1px solid #ccc; border-radius: 4px; font-family: monospace;"
    />
    <small style="color: #666; font-size: 0.85em;">
      Format OpenRouter : <code>provider/model-name</code> (ex: openai/gemini-2.5-flash, anthropic/claude-3.5-sonnet)
    </small>
  </div>
  <!-- ↑↑↑ FIN NOUVEAU CHAMP ↑↑↑ -->

  <form id="initialChunkUploadForm" enctype="multipart/form-data" class="stage-upload-form">
    <input type="file" id="initialChunkUploadInput" name="file" accept=".csv" />
    <button type="submit">Upload output.csv</button>
  </form>
  <button id="runInitialChunkBtn">Generate Chunks</button>
  <div id="initialChunkResult" class="result"></div>
</section>
```

---

#### 4.2 Ajouter le modal de confirmation fallback

**Localisation** : Après le modal Settings (ligne 122), avant la section `<script>`

```html
<!-- Modal de confirmation fallback OpenRouter -->
<div id="fallbackModal" class="modal" style="display: none;">
  <div class="modal-content" style="max-width: 500px;">
    <h2 style="color: #d32f2f; margin-top: 0;">⚠️ OpenRouter inaccessible</h2>
    <p id="fallbackErrorMessage" style="margin: 15px 0; padding: 10px; background: #fff3cd; border-left: 4px solid #ffc107; color: #856404;">
      <!-- Le message d'erreur sera injecté ici par JavaScript -->
    </p>
    <p style="margin: 15px 0;">
      Voulez-vous utiliser <strong>OpenAI (gpt-4o-mini)</strong> à la place ?
    </p>
    <div style="display: flex; gap: 10px; justify-content: flex-end; margin-top: 20px;">
      <button id="cancelFallback" style="background: #6c757d; color: white; padding: 10px 20px; border: none; border-radius: 4px; cursor: pointer;">
        Annuler
      </button>
      <button id="confirmFallback" style="background: #1976d2; color: white; padding: 10px 20px; border: none; border-radius: 4px; cursor: pointer;">
        Oui, utiliser OpenAI
      </button>
    </div>
  </div>
</div>

<!-- Style du modal -->
<style>
.modal {
  position: fixed;
  z-index: 9999;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  overflow: auto;
  background-color: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal-content {
  background-color: #fefefe;
  padding: 20px;
  border: 1px solid #888;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
}
</style>
```

---

#### 4.3 Ajouter OpenRouter dans le modal Settings

**Localisation** : Ligne 87-122, dans le formulaire Settings

**Ajouter après la section OpenAI** (ligne ~95) :

```html
<!-- Section OpenAI existante -->
<h3 style="margin-top:0; margin-bottom: 10px; color: #1976d2;">OpenAI</h3>
<label for="openaiKey">API Key</label>
<input type="text" id="openaiKey" name="OPENAI_API_KEY" autocomplete="off" placeholder="sk-..." />

<!-- ↓↓↓ NOUVELLE SECTION OPENROUTER ↓↓↓ -->
<h3 style="margin-top: 20px; margin-bottom: 10px; color: #1976d2;">OpenRouter</h3>
<label for="openrouterKey">API Key</label>
<input type="text" id="openrouterKey" name="OPENROUTER_API_KEY" autocomplete="off" placeholder="sk-or-v1-..." />

<label for="openrouterModel" style="margin-top: 10px;">Modèle par défaut</label>
<input
  type="text"
  id="openrouterModel"
  name="OPENROUTER_DEFAULT_MODEL"
  autocomplete="off"
  placeholder="openai/gemini-2.5-flash"
  style="font-family: monospace;"
/>
<small style="color: #666; display: block; margin-top: 5px;">
  Format : provider/model-name
</small>
<!-- ↑↑↑ FIN NOUVELLE SECTION ↑↑↑ -->

<!-- Section Pinecone (existante) -->
<h3 style="margin-top: 20px; margin-bottom: 10px; color: #1976d2;">Pinecone</h3>
<!-- ... reste du code ... -->
```

---

#### 4.4 JavaScript : Charger les credentials OpenRouter

**Localisation** : Fonction `loadCredentials()` (ligne 224-269)

**Modification** : Ajouter le chargement des clés OpenRouter

```javascript
// Ligne ~247, après le chargement de openaiKey
if (data.OPENROUTER_API_KEY) {
  const maskedOR = data.OPENROUTER_API_KEY.substring(0, 20) + '######';
  document.getElementById('openrouterKey').value = maskedOR;
  document.getElementById('openrouterKey').dataset.fullKey = data.OPENROUTER_API_KEY;
}

if (data.OPENROUTER_DEFAULT_MODEL) {
  document.getElementById('openrouterModel').value = data.OPENROUTER_DEFAULT_MODEL;
}
```

---

#### 4.5 JavaScript : Sauvegarder les credentials OpenRouter

**Localisation** : Fonction de soumission du formulaire Settings (ligne 285-322)

**Modification** : Inclure les champs OpenRouter dans FormData

```javascript
// Ligne ~308, avant le fetch
const openrouterKeyInput = document.getElementById('openrouterKey');
let openrouterKey = openrouterKeyInput.value;

// Si la clé affichée est masquée, utiliser la clé complète stockée
if (openrouterKey.includes('######') && openrouterKeyInput.dataset.fullKey) {
  openrouterKey = openrouterKeyInput.dataset.fullKey;
}

const openrouterModel = document.getElementById('openrouterModel').value;

// Ajouter à FormData
formData.append('OPENROUTER_API_KEY', openrouterKey);
formData.append('OPENROUTER_DEFAULT_MODEL', openrouterModel);
```

---

#### 4.6 JavaScript : Gestion du bouton "Generate Chunks"

**Localisation** : Event listener `#runInitialChunkBtn` (chercher dans le code, généralement après ligne 450)

**Remplacer l'ancien code** par :

```javascript
document.getElementById('runInitialChunkBtn').addEventListener('click', async function() {
  const resultDiv = document.getElementById('initialChunkResult');
  const modelInput = document.getElementById('chunkingModel').value.trim();

  if (!modelInput) {
    resultDiv.innerHTML = '<p style="color: red;">⚠️ Veuillez spécifier un modèle</p>';
    return;
  }

  resultDiv.innerHTML = '<p>⏳ Chunking en cours (peut prendre plusieurs minutes)...</p>';

  try {
    // Créer FormData avec le modèle
    const formData = new FormData();
    formData.append('model', modelInput);

    const response = await fetch('/initial_text_chunking', {
      method: 'POST',
      body: formData
    });

    const data = await response.json();

    // Vérifier si un fallback est nécessaire
    if (data.fallback_needed) {
      // Afficher le modal de confirmation
      showFallbackModal(data.error);
    } else if (data.success) {
      resultDiv.innerHTML = '<p style="color: green;">✓ Chunking initial réussi avec OpenRouter !</p>';
      // Activer la section suivante
      document.getElementById('dense-embedding-section').style.display = 'block';
    } else {
      resultDiv.innerHTML = `<p style="color: red;">✗ Erreur : ${data.message || 'Erreur inconnue'}</p>`;
    }
  } catch (error) {
    resultDiv.innerHTML = `<p style="color: red;">✗ Erreur réseau : ${error.message}</p>`;
  }
});

// Fonction pour afficher le modal de fallback
function showFallbackModal(errorMessage) {
  const modal = document.getElementById('fallbackModal');
  const errorMsgElement = document.getElementById('fallbackErrorMessage');

  errorMsgElement.textContent = errorMessage || "Impossible de se connecter à OpenRouter";
  modal.style.display = 'flex';
}

// Bouton "Confirmer" dans le modal
document.getElementById('confirmFallback').addEventListener('click', async function() {
  const modal = document.getElementById('fallbackModal');
  const resultDiv = document.getElementById('initialChunkResult');

  modal.style.display = 'none';
  resultDiv.innerHTML = '<p>⏳ Chunking avec OpenAI (fallback) en cours...</p>';

  try {
    const response = await fetch('/initial_text_chunking_fallback', {
      method: 'POST'
    });

    const data = await response.json();

    if (data.success) {
      resultDiv.innerHTML = '<p style="color: green;">✓ Chunking réussi avec OpenAI (fallback) !</p>';
      document.getElementById('dense-embedding-section').style.display = 'block';
    } else {
      resultDiv.innerHTML = `<p style="color: red;">✗ Erreur fallback : ${data.message || 'Erreur inconnue'}</p>`;
    }
  } catch (error) {
    resultDiv.innerHTML = `<p style="color: red;">✗ Erreur réseau : ${error.message}</p>`;
  }
});

// Bouton "Annuler" dans le modal
document.getElementById('cancelFallback').addEventListener('click', function() {
  const modal = document.getElementById('fallbackModal');
  const resultDiv = document.getElementById('initialChunkResult');

  modal.style.display = 'none';
  resultDiv.innerHTML = '<p style="color: orange;">⚠️ Chunking annulé par l\'utilisateur</p>';
});
```

---

## 🧪 Tests & Validation

### Tests unitaires (à créer)

**Fichier** : `tests/test_openrouter_chunking.py`

```python
import unittest
from scripts.rad_chunk import gpt_recode_batch_with_openrouter
from unittest.mock import patch, MagicMock

class TestOpenRouterChunking(unittest.TestCase):

    @patch('scripts.rad_chunk.client_openrouter')
    def test_openrouter_success(self, mock_client):
        """Test appel réussi vers OpenRouter"""
        # Mock response
        mock_response = MagicMock()
        mock_response.choices[0].message.content = "Texte nettoyé"
        mock_client.chat.completions.create.return_value = mock_response

        success, texts, error = gpt_recode_batch_with_openrouter(
            ["Texte brut OCR"],
            "Instructions de nettoyage"
        )

        self.assertTrue(success)
        self.assertEqual(texts[0], "Texte nettoyé")
        self.assertIsNone(error)

    @patch('scripts.rad_chunk.client_openrouter')
    def test_openrouter_failure(self, mock_client):
        """Test échec OpenRouter (retourne erreur)"""
        mock_client.chat.completions.create.side_effect = Exception("API Error")

        success, texts, error = gpt_recode_batch_with_openrouter(
            ["Texte brut"],
            "Instructions"
        )

        self.assertFalse(success)
        self.assertIsNone(texts)
        self.assertIn("API Error", error)

    def test_openrouter_client_not_initialized(self):
        """Test si client OpenRouter non initialisé"""
        # À tester avec OPENROUTER_API_KEY vide
        pass
```

---

### Tests d'intégration

#### Scénario 1 : OpenRouter accessible
1. Configurer `.env` avec `OPENROUTER_API_KEY` valide
2. Uploader `output.csv` dans l'UI
3. Entrer `openai/gemini-2.5-flash` dans le champ modèle
4. Cliquer "Generate Chunks"
5. **Attendu** : Chunking réussi, message vert, section 3.2 activée

#### Scénario 2 : OpenRouter inaccessible (clé invalide)
1. Configurer `.env` avec `OPENROUTER_API_KEY` invalide
2. Uploader `output.csv`
3. Cliquer "Generate Chunks"
4. **Attendu** : Modal de confirmation affiché avec message d'erreur
5. Cliquer "Oui, utiliser OpenAI"
6. **Attendu** : Chunking réussi avec OpenAI, message vert avec "(fallback)"

#### Scénario 3 : Annulation fallback
1. Simuler erreur OpenRouter
2. Modal affiché
3. Cliquer "Annuler"
4. **Attendu** : Message orange "Chunking annulé", section 3.2 reste cachée

#### Scénario 4 : Modèle personnalisé
1. Entrer `anthropic/claude-3.5-sonnet` dans le champ modèle
2. Lancer chunking
3. **Attendu** : Utilise Claude au lieu de Gemini

---

### Tests manuels (checklist)

- [ ] **Clé OpenRouter manquante** : Vérifier fallback immédiat sans modal
- [ ] **Timeout réseau** : Simuler avec `--timeout 1` → modal affiché
- [ ] **Quota OpenRouter dépassé** : Tester avec compte gratuit saturé
- [ ] **Logs** : Vérifier que `chunking.log` distingue OpenRouter/OpenAI
- [ ] **Console** : Messages `✓ OpenRouter` ou `→ Utilisation OpenAI` affichés
- [ ] **Persistence** : Recharger la page → clé OpenRouter masquée correctement

---

## 📖 Documentation à mettre à jour

### 1. README.md

**Section à ajouter/modifier** : "Configuration"

```markdown
### Variables d'environnement

Créez un fichier `.env` à la racine du projet avec les clés suivantes :

```bash
# OpenAI (pour embeddings)
OPENAI_API_KEY=sk-...

# OpenRouter (pour chunking/recodage de texte)
OPENROUTER_API_KEY=sk-or-v1-...
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1
OPENROUTER_DEFAULT_MODEL=openai/gemini-2.5-flash

# Autres services (Pinecone, Mistral, etc.)
# ...
```

**Comportement du chunking** :
- Par défaut, le système utilise **OpenRouter avec Gemini 2.5 Flash** pour le recodage de texte
- Si OpenRouter est inaccessible (erreur réseau, clé invalide, quota dépassé), le système demande confirmation pour basculer sur **OpenAI (gpt-4o-mini)**
- Les embeddings continuent d'utiliser **OpenAI (text-embedding-3-large)**

**Modèles OpenRouter supportés** :
- `openai/gemini-2.5-flash` (recommandé, par défaut)
- `anthropic/claude-3.5-sonnet`
- `openai/gpt-4o`
- Voir la liste complète : https://openrouter.ai/models
```

---

### 2. .claude/AGENTS.md

**Section à modifier** : "Agent `rad_chunk.py`" (ligne 92-135)

**Modifications** :

1. **Ligne 101** : Ajouter OpenRouter dans les dépendances

```markdown
### Dépendances & environnement
- `OPENAI_API_KEY` obligatoire pour embeddings (text-embedding-3-large)
- `OPENROUTER_API_KEY` optionnel pour recodage GPT (Gemini 2.5 Flash via OpenRouter). Si absent, fallback sur OpenAI.
- Librairies: `langchain_text_splitters`, `openai`, `spacy` (`fr_core_news_md` téléchargé si absent), `tqdm`, `pandas`.
```

2. **Ligne 108** : Documenter le nouveau paramètre `--model`

```markdown
### Paramètres CLI
```bash
python scripts/rad_chunk.py \
  --input sources/MaBiblio/output.csv \
  --output sources/MaBiblio \
  --phase all \
  --model openai/gemini-2.5-flash  # Optionnel, défaut: OPENROUTER_DEFAULT_MODEL
```
- `--input` : CSV (phase `initial`) ou JSON (phases `dense`/`sparse`).
- `--output` : dossier cible des JSON (créé si besoin).
- `--phase` : `initial`, `dense`, `sparse`, ou `all` (enchaîne les trois).
- `--model` : Modèle OpenRouter pour le recodage (format `provider/model-name`). Si absent, utilise `OPENROUTER_DEFAULT_MODEL` du .env, sinon fallback OpenAI.
```

3. **Ligne 119** : Documenter le comportement OpenRouter

```markdown
### Détails par phase
- **initial** :
  - Lit un CSV, découpe le champ `texteocr` en chunks (~2 500 tokens avec chevauchement 250)
  - **Recodage via OpenRouter** (Gemini 2.5 Flash par défaut) si `OPENROUTER_API_KEY` est configurée
  - **Fallback OpenAI** (gpt-4o-mini) si OpenRouter échoue ou si l'utilisateur annule
  - Saute le recodage si l'OCR provient de Mistral (chunks Markdown déjà propres)
  - Sauvegarde `output_chunks.json`
- **dense** : attend un fichier `_chunks.json`, génère les embeddings denses OpenAI (`text-embedding-3-large`), écrit `_chunks_with_embeddings.json`.
- **sparse** : attend `_chunks_with_embeddings.json`, dérive les features spaCy (POS filtrés, lemmas, TF normalisé, hachage mod 100 000), sauvegarde `_chunks_with_embeddings_sparse.json`.
- **all** : enchaîne les trois sous-étapes avec journalisation dans `<output>/chunking.log`.
```

4. **Ligne 130** : Ajouter section troubleshooting

```markdown
### Comportement complémentaire
- Si la clé OpenAI est absente, le script la demande et propose de la stocker via `python-dotenv`.
- **Mécanisme de fallback OpenRouter → OpenAI** :
  1. Le script tente d'utiliser OpenRouter en premier
  2. En cas d'erreur (réseau, authentification, quota), une exception `OPENROUTER_UNAVAILABLE` est levée
  3. L'interface web affiche un modal de confirmation
  4. Si l'utilisateur confirme, le chunking est relancé avec OpenAI
  5. Si l'utilisateur annule, le processus s'arrête
- SpaCy : tronque les textes très longs à `nlp.max_length` (ou 50 000 caractères) pour éviter les dépassements.
- Les identifiants de chunk incluent `doc_id`, `chunk_index`, `total_chunks` pour faciliter l'upload.
- Les erreurs d'API GPT sont réessayées séquentiellement (seconde passe) avant fallback sur le texte brut.
```

---

### 3. Nouveau fichier : .claude/OPENROUTER.md

**Créer un nouveau fichier** avec le contenu suivant :

```markdown
# Guide OpenRouter pour RAGpy

## Qu'est-ce qu'OpenRouter ?

OpenRouter est un service qui fournit un **accès unifié à plusieurs modèles de langage** (OpenAI, Anthropic, Google, Meta, etc.) via une seule API.

**Avantages** :
- Un seul compte pour accéder à Gemini, Claude, GPT-4, Llama, etc.
- Tarification compétitive (souvent moins cher que les APIs officielles)
- Basculement facile entre modèles sans changer de code
- Pas besoin de comptes séparés chez chaque fournisseur

---

## Configuration dans RAGpy

### 1. Créer un compte OpenRouter

1. Aller sur https://openrouter.ai/
2. S'inscrire (gratuit)
3. Aller dans "Keys" → "Create New Key"
4. Copier la clé (format : `sk-or-v1-...`)

### 2. Ajouter la clé dans RAGpy

**Option A : Fichier .env**
```bash
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxx
OPENROUTER_DEFAULT_MODEL=openai/gemini-2.5-flash
```

**Option B : Interface web**
1. Lancer RAGpy : `uvicorn app.main:app --reload --host 0.0.0.0 --port 8000`
2. Ouvrir http://localhost:8000
3. Cliquer sur "Settings" (⚙️)
4. Remplir "OpenRouter API Key"
5. Cliquer "Save"

---

## Modèles recommandés pour le chunking

| Modèle | Format OpenRouter | Coût (par 1M tokens) | Performance |
|--------|-------------------|----------------------|-------------|
| **Gemini 2.5 Flash** | `openai/gemini-2.5-flash` | ~$0.15 | ⭐⭐⭐⭐⭐ Recommandé |
| Claude 3.5 Sonnet | `anthropic/claude-3.5-sonnet` | ~$3.00 | ⭐⭐⭐⭐⭐ Très bon |
| GPT-4o mini | `openai/gpt-4o-mini` | ~$0.30 | ⭐⭐⭐⭐ Bon |
| Llama 3.3 70B | `meta-llama/llama-3.3-70b-instruct` | ~$0.50 | ⭐⭐⭐ Moyen |

**Pour le chunking de texte OCR**, Gemini 2.5 Flash est le meilleur compromis coût/qualité.

---

## Utilisation dans l'interface

### Chunking avec OpenRouter

1. Uploader `output.csv` dans la section 3.1
2. Le champ "Modèle de recodage" est pré-rempli avec `openai/gemini-2.5-flash`
3. (Optionnel) Changer le modèle si souhaité
4. Cliquer "Generate Chunks"

**Le système tentera d'utiliser OpenRouter automatiquement.**

### Si OpenRouter est inaccessible

Un modal s'affichera :
```
⚠️ OpenRouter inaccessible
[Message d'erreur]

Voulez-vous utiliser OpenAI (gpt-4o-mini) à la place ?

[Annuler]  [Oui, utiliser OpenAI]
```

- **Oui** → Le chunking continue avec OpenAI
- **Annuler** → Le processus s'arrête

---

## Troubleshooting

### Erreur : "OPENROUTER_API_KEY non configurée"

**Solution** : Ajouter la clé dans `.env` ou via Settings

### Erreur : "Unauthorized" / HTTP 401

**Causes possibles** :
- Clé API invalide
- Clé expirée
- Crédit OpenRouter épuisé

**Solution** :
1. Vérifier la clé sur https://openrouter.ai/keys
2. Vérifier le solde du compte
3. Regénérer une nouvelle clé si nécessaire

### Erreur : "Rate limit exceeded" / HTTP 429

**Cause** : Quota OpenRouter dépassé (trop de requêtes)

**Solution** :
- Attendre quelques minutes
- Augmenter le crédit du compte
- Utiliser le fallback OpenAI

### Erreur : "Model not found" / HTTP 404

**Cause** : Format de modèle invalide

**Solution** : Vérifier le format sur https://openrouter.ai/models
- Correct : `openai/gemini-2.5-flash`
- Incorrect : `gemini-2.5-flash` (manque le provider)

### Timeout / Erreur réseau

**Cause** : Connexion OpenRouter lente ou indisponible

**Solution** :
- Vérifier la connexion internet
- Utiliser le fallback OpenAI
- Vérifier le statut : https://status.openrouter.ai/

---

## Comparaison de coûts

### Recodage de 1000 chunks (moyenne 500 tokens par chunk)

| Fournisseur | Modèle | Coût estimé |
|-------------|--------|-------------|
| **OpenRouter** | Gemini 2.5 Flash | **~$0.08** |
| OpenRouter | Claude 3.5 Sonnet | ~$1.50 |
| OpenRouter | GPT-4o mini | ~$0.15 |
| **OpenAI direct** | gpt-4o-mini | ~$0.20 |

**Économie avec OpenRouter (Gemini)** : ~60% par rapport à OpenAI direct

---

## Logs et débogage

Les logs du chunking incluent des messages distincts :

**Succès OpenRouter** :
```
✓ Client OpenRouter initialisé (modèle par défaut : openai/gemini-2.5-flash)
→ Tentative de recodage via OpenRouter (openai/gemini-2.5-flash)...
✓ Recodage OpenRouter réussi (150 chunks)
```

**Fallback OpenAI** :
```
✗ Erreur globale OpenRouter: Unauthorized
→ Utilisation d'OpenAI (gpt-4o-mini) pour le recodage...
```

Consultez les logs :
- Console : stdout en temps réel
- Fichier : `uploads/<session>/chunking.log`

---

## Ressources

- Documentation officielle : https://openrouter.ai/docs
- Liste des modèles : https://openrouter.ai/models
- Pricing : https://openrouter.ai/models?pricing=true
- Status : https://status.openrouter.ai/
```

---

## 📦 Checklist post-développement

### Configuration
- [ ] Créer fichier `.env.example` avec :
  ```bash
  OPENAI_API_KEY=sk-...
  OPENROUTER_API_KEY=sk-or-v1-...
  OPENROUTER_DEFAULT_MODEL=openai/gemini-2.5-flash
  ```
- [ ] Vérifier `.gitignore` inclut `.env`

### Code
- [ ] Tester avec `.env` minimal (sans OPENROUTER_API_KEY)
  - **Attendu** : Fallback immédiat sur OpenAI
- [ ] Tester avec OPENROUTER_API_KEY valide
  - **Attendu** : Utilisation d'OpenRouter
- [ ] Tester avec OPENROUTER_API_KEY invalide
  - **Attendu** : Modal de confirmation, puis fallback sur OpenAI si confirmé

### Logs
- [ ] Vérifier que les logs distinguent OpenRouter/OpenAI
- [ ] Console affiche `✓ OpenRouter` ou `→ OpenAI` clairement
- [ ] Fichier `chunking.log` contient les messages d'erreur complets

### Documentation
- [ ] README.md mis à jour avec variables OpenRouter
- [ ] AGENTS.md mis à jour avec comportement fallback
- [ ] OPENROUTER.md créé avec guide complet
- [ ] Ajouter section "Migration" dans README si nécessaire

### Interface
- [ ] Champ modèle pré-rempli avec `openai/gemini-2.5-flash`
- [ ] Modal fallback s'affiche correctement
- [ ] Boutons "Confirmer" / "Annuler" fonctionnent
- [ ] Settings permet de sauvegarder/charger la clé OpenRouter

### Tests
- [ ] Tests unitaires passent (si créés)
- [ ] Tests d'intégration pour les 4 scénarios
- [ ] Validation manuelle des erreurs (timeout, quota, etc.)

### Déploiement
- [ ] Redémarrer le serveur : `uvicorn app.main:app --reload`
- [ ] Tester un chunking complet de bout en bout
- [ ] Vérifier que les fichiers JSON produits sont identiques (OpenRouter vs OpenAI)

---

## 🎯 Ordre d'implémentation recommandé

### Phase 1 : Backend (rad_chunk.py)
1. ✅ Ajouter variables .env (OPENROUTER_API_KEY, etc.)
2. ✅ Créer client OpenRouter (après ligne 70)
3. ✅ Implémenter `gpt_recode_batch_with_openrouter()`
4. ✅ Ajouter argument `--model` au parser CLI
5. ✅ Modifier `process_document_chunks()` pour utiliser OpenRouter
6. ✅ Modifier `main()` pour passer le modèle

### Phase 2 : Backend (app/main.py)
7. ✅ Créer endpoint `/check_openrouter`
8. ✅ Modifier `/initial_text_chunking` (ajouter paramètre `model`, gérer erreur)
9. ✅ Créer endpoint `/initial_text_chunking_fallback`
10. ✅ Modifier `/get_credentials` (ajouter OpenRouter)
11. ✅ Modifier `/save_credentials` (sauvegarder OpenRouter)

### Phase 3 : Frontend (index.html)
12. ✅ Ajouter champ modèle dans section 3.1
13. ✅ Créer modal de confirmation fallback
14. ✅ Ajouter OpenRouter dans Settings
15. ✅ JavaScript : charger credentials OpenRouter
16. ✅ JavaScript : sauvegarder credentials OpenRouter
17. ✅ JavaScript : gérer bouton "Generate Chunks" + modal

### Phase 4 : Tests & Documentation
18. ✅ Tests unitaires (optionnel mais recommandé)
19. ✅ Tests d'intégration (4 scénarios)
20. ✅ Mettre à jour README.md
21. ✅ Mettre à jour AGENTS.md
22. ✅ Créer OPENROUTER.md

### Phase 5 : Validation finale
23. ✅ Checklist post-dev complète
24. ✅ Test de bout en bout (upload CSV → chunking → embeddings → DB)
25. ✅ Vérification des logs et console

---

## 📝 Notes importantes

### Format des modèles OpenRouter

OpenRouter utilise le format `provider/model-name` :
- ✅ Correct : `openai/gemini-2.5-flash`
- ✅ Correct : `anthropic/claude-3.5-sonnet`
- ❌ Incorrect : `gemini-2.5-flash` (manque le provider)
- ❌ Incorrect : `gpt-4o-mini` (ce n'est pas un modèle OpenRouter, mais OpenAI direct)

### Headers requis pour OpenRouter

L'API OpenRouter requiert des headers spécifiques :
```python
extra_headers={
    "HTTP-Referer": "https://ragpy.local",  # Obligatoire
    "X-Title": "RAGpy Chunking Pipeline"     # Optionnel mais recommandé
}
```

### Gestion du fallback

**Déclencheurs de fallback** (toute erreur OpenRouter) :
- Erreurs réseau (timeout, connexion refusée)
- Erreurs HTTP 4xx (authentification, quota)
- Erreurs HTTP 5xx (serveur OpenRouter down)
- Exceptions Python (malformation requête, etc.)

**Processus** :
1. Erreur détectée → exception `OPENROUTER_UNAVAILABLE` levée
2. Backend FastAPI capture l'exception → retourne `{fallback_needed: true}`
3. Frontend affiche modal de confirmation
4. Si confirmé → appel `/initial_text_chunking_fallback` (sans `--model`)
5. Si annulé → processus stoppé

### Compatibilité embeddings

**Les embeddings continuent d'utiliser OpenAI** :
- Modèle : `text-embedding-3-large`
- Fonction : `get_embeddings_batch()` (ligne 301 de rad_chunk.py)
- **Aucune modification nécessaire** pour cette partie

---

## 🔍 Points de vigilance

### Sécurité
- Ne jamais commit `.env` dans Git
- Masquer les clés API dans l'UI (afficher seulement les 20 premiers caractères)
- Valider les inputs utilisateur (modèle, clés API)

### Performance
- OpenRouter peut être plus lent qu'OpenAI direct (latence réseau supplémentaire)
- Prévoir des timeouts appropriés (actuellement 3600s = 1h)
- Logger les temps de réponse pour comparaison

### Coûts
- Vérifier les prix sur https://openrouter.ai/models?pricing=true
- Gemini 2.5 Flash est généralement le moins cher (~$0.15/1M tokens)
- Comparer avec OpenAI direct (gpt-4o-mini ~$0.30/1M tokens)

### UX
- Expliquer clairement le fallback à l'utilisateur
- Afficher les messages d'erreur de manière compréhensible
- Permettre l'annulation à tout moment

---

## ✅ Critères de succès

Le développement sera considéré comme réussi si :

1. **Fonctionnel** :
   - [ ] Chunking avec OpenRouter fonctionne (succès)
   - [ ] Fallback vers OpenAI fonctionne (en cas d'erreur)
   - [ ] Modal de confirmation s'affiche et fonctionne
   - [ ] Annulation du processus fonctionne

2. **Configuration** :
   - [ ] Variables .env chargées correctement
   - [ ] Settings UI permet de configurer OpenRouter
   - [ ] Clés masquées dans l'UI mais sauvegardées en entier

3. **Robustesse** :
   - [ ] Gère les erreurs réseau (timeout, connexion)
   - [ ] Gère les erreurs API (401, 429, 500)
   - [ ] Logs clairs et exploitables
   - [ ] Pas de régression sur les embeddings (toujours OpenAI)

4. **Documentation** :
   - [ ] README.md à jour
   - [ ] AGENTS.md à jour
   - [ ] OPENROUTER.md créé et complet
   - [ ] Code commenté (surtout les parties critiques)

5. **Tests** :
   - [ ] Les 4 scénarios d'intégration passent
   - [ ] Tests manuels concluants
   - [ ] Validation de bout en bout (CSV → chunks → embeddings → DB)

---

## 📞 Support

En cas de problème :
1. Consulter `chunking.log` dans le dossier de sortie
2. Vérifier les logs du serveur FastAPI (console ou `ragpy_server.log`)
3. Tester la clé OpenRouter sur https://openrouter.ai/playground
4. Vérifier le statut d'OpenRouter : https://status.openrouter.ai/

---

**Fin du plan de développement**
