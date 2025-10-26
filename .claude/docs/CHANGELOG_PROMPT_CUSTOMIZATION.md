# Changelog : Personnalisation du Prompt Zotero

## 🎯 Objectif

Permettre la personnalisation facile du prompt de génération de fiches de lecture Zotero **sans modifier le code Python**, via un simple fichier Markdown éditable.

---

## ✨ Nouveautés

### 1. Fichier de Template de Prompt

**Fichier** : `app/utils/zotero_prompt.md`

- Template Markdown avec placeholders `{VARIABLE}`
- Rechargement automatique à chaque génération (pas de redémarrage serveur)
- Encodage UTF-8 pour support multilingue

**Placeholders disponibles** :
- `{TITLE}` - Titre de l'article
- `{AUTHORS}` - Auteurs
- `{DATE}` - Date de publication
- `{DOI}` - Digital Object Identifier
- `{URL}` - URL de l'article
- `{ABSTRACT}` - Résumé
- `{TEXT}` - Texte complet OCR (limité à 8000 caractères)
- `{LANGUAGE}` - Langue cible (français, English, español, etc.)

### 2. Système de Chargement Dynamique

**Modifications** : `app/utils/llm_note_generator.py`

Nouvelles fonctions :
- `_load_prompt_template()` : Charge le template depuis le fichier
- `_build_prompt()` : Remplace les placeholders par les vraies valeurs

**Robustesse** :
- Fallback automatique vers prompt hardcodé si fichier introuvable
- Logging détaillé (succès/erreurs)
- Gestion d'erreurs gracieuse

### 3. Documentation Complète

**Fichier** : `app/utils/README_ZOTERO_PROMPT.md` (130+ lignes)

Contient :
- Vue d'ensemble du système
- Liste complète des placeholders
- 3 exemples de personnalisation :
  - Fiche minimaliste (100 mots)
  - Fiche détaillée pour thèse (500 mots)
  - Fiche orientée applications techniques
- Bonnes pratiques et pièges à éviter
- Guide de dépannage
- Workflow de modification

### 4. Tests de Validation

**Fichier** : `tests/test_prompt_file.py`

5 tests automatisés :
1. ✅ Existence du fichier
2. ✅ Lisibilité (encodage UTF-8)
3. ✅ Présence des 8 placeholders requis
4. ✅ Structure valide (markdown, HTML, instructions)
5. ✅ Remplacement fonctionnel des placeholders

**Exécution** :
```bash
python tests/test_prompt_file.py
```

### 5. Mise à Jour de la Documentation

**Fichier** : `README.md`

Nouvelle section : "Personnalisation du Prompt"
- Explication du mécanisme
- Exemple de modification rapide
- Lien vers le guide complet

---

## 🔧 Architecture Technique

### Flux de Génération

```
1. Utilisateur clique "Generate Zotero Notes"
   ↓
2. Backend : /generate_zotero_notes endpoint
   ↓
3. Pour chaque article :
   ├─ Extraction métadonnées depuis output.csv
   ├─ Appel llm_note_generator.build_note_html()
   │  ├─ _load_prompt_template() → Charge zotero_prompt.md
   │  ├─ _build_prompt() → Remplace {PLACEHOLDERS}
   │  └─ _generate_with_llm() → Envoie au LLM
   ├─ Vérification idempotence (sentinel)
   └─ Création note Zotero via API
```

### Gestion des Erreurs

| Erreur | Comportement |
|--------|-------------|
| Fichier `zotero_prompt.md` absent | Fallback vers prompt hardcodé + warning log |
| Encodage invalide | Exception capturée → fallback + erreur log |
| Placeholder manquant | Reste inchangé dans le prompt final |
| Fichier corrompu | Fallback automatique |

---

## 📊 Avantages

### Pour l'Utilisateur Final

✅ **Personnalisation sans code** : Édition d'un simple fichier Markdown
✅ **Rechargement automatique** : Pas de redémarrage serveur
✅ **Multilingue** : Support via placeholder `{LANGUAGE}`
✅ **Versionnable** : Fichier texte simple → Git-friendly
✅ **Réversible** : Fallback automatique en cas d'erreur

### Pour le Développeur

✅ **Découplage** : Prompt séparé de la logique métier
✅ **Maintenabilité** : Modifications prompt sans toucher au code
✅ **Testabilité** : Tests automatisés de validation
✅ **Extensibilité** : Ajout facile de nouveaux placeholders

---

## 🚀 Utilisation

### Modification Basique

1. Ouvrir `app/utils/zotero_prompt.md`
2. Modifier le texte (garder les `{PLACEHOLDERS}`)
3. Sauvegarder
4. Générer une fiche → changements appliqués immédiatement

### Exemple : Fiche Ultra-Courte

```markdown
Résume {TITLE} par {AUTHORS} ({DATE}) en 50 mots en {LANGUAGE}.

Texte : {TEXT}

Format HTML.
```

### Exemple : Fiche Critique

```markdown
Analyse critique de {TITLE} en {LANGUAGE}.

Auteurs : {AUTHORS}
Texte : {TEXT}

1. Forces de l'article (3 points)
2. Faiblesses et limites (3 points)
3. Contribution à mon domaine de recherche

HTML, 300 mots max.
```

---

## ✅ Tests

### Test Automatique

```bash
cd ragpy
python tests/test_prompt_file.py
```

**Sortie attendue** :
```
======================================================================
ZOTERO PROMPT FILE VALIDATION TESTS
======================================================================

✅ Prompt file exists
✅ Prompt file readable, length: 1136 characters
✅ All 8 placeholders found
✅ 4/4 structure checks passed
✅ All placeholders replaced successfully

======================================================================
🎉 SUCCESS: All 5 tests passed!
======================================================================
```

### Test Manuel

1. Modifier `app/utils/zotero_prompt.md`
2. Upload export Zotero (1 article test)
3. Pipeline jusqu'à "Sparse Embeddings"
4. Générer note Zotero
5. Vérifier dans Zotero Desktop que la structure correspond

---

## 📝 Exemples de Cas d'Usage

### Thèse de Doctorat

Prompt long (500 mots) avec sections :
- Contexte scientifique
- Positionnement dans la littérature
- Critique méthodologique
- Pertinence pour ma thèse

### Veille Technologique

Prompt technique avec focus sur :
- Technologies/frameworks utilisés
- Benchmarks et performances
- Code disponible (GitHub)
- Reproductibilité

### Revue de Littérature

Prompt comparatif :
- Approche (théorique/empirique)
- Comparaison avec travaux similaires
- Classification thématique

---

## 🐛 Problèmes Connus

### Encodage

**Symptôme** : Caractères mal encodés (é → Ã©)
**Solution** : Recréer le fichier en UTF-8 :
```bash
cat > app/utils/zotero_prompt.md << 'EOF'
[votre prompt ici]
EOF
```

### Placeholders Non Remplacés

**Symptôme** : `{TITLE}` apparaît tel quel dans la note
**Cause** : Faute de frappe ou espace (`{ TITLE }`)
**Solution** : Vérifier l'orthographe exacte (majuscules, pas d'espaces)

---

## 🔮 Évolutions Futures

### Court Terme

- [ ] Interface UI pour éditer le prompt (éditeur intégré)
- [ ] Galerie de templates pré-configurés (académique, technique, vulgarisation)
- [ ] Validation syntaxique du prompt (détection placeholders manquants)

### Moyen Terme

- [ ] Support de variables conditionnelles (`{IF_DOI_EXISTS}...{ENDIF}`)
- [ ] Templates multiples (sélection par domaine/type d'article)
- [ ] Prévisualisation du prompt avec métadonnées réelles

### Long Terme

- [ ] Éditeur WYSIWYG dans l'interface web
- [ ] Marketplace de templates communautaires
- [ ] A/B testing automatique de prompts (qualité des fiches)

---

## 📚 Références

- [app/utils/zotero_prompt.md](app/utils/zotero_prompt.md) - Template actuel
- [app/utils/README_ZOTERO_PROMPT.md](app/utils/README_ZOTERO_PROMPT.md) - Guide complet
- [app/utils/llm_note_generator.py](app/utils/llm_note_generator.py) - Code de chargement
- [tests/test_prompt_file.py](tests/test_prompt_file.py) - Tests de validation

---

## 👥 Contributeurs

- Implémentation initiale : Claude Code Agent
- Tests et validation : Tests automatisés
- Documentation : README + Guide utilisateur

---

**Date de Release** : 2025-10-26
**Version** : 1.1.0 (Feature: Custom Prompt)
