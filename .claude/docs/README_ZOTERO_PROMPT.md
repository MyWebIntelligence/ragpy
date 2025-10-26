# Personnalisation du Prompt de Génération de Fiches Zotero

## 📝 Vue d'ensemble

Le fichier [zotero_prompt.md](zotero_prompt.md) contient le template du prompt utilisé pour générer les fiches de lecture via LLM. Vous pouvez le modifier pour personnaliser le style, la structure et le contenu des fiches générées.

## 🔧 Comment ça fonctionne

1. **Chargement** : Le système charge automatiquement `zotero_prompt.md` à chaque génération de fiche
2. **Remplacement** : Les placeholders `{VARIABLE}` sont remplacés par les vraies valeurs
3. **Génération** : Le prompt complet est envoyé au LLM (GPT, Claude, Gemini, etc.)

## 📋 Placeholders Disponibles

Vous pouvez utiliser ces placeholders dans votre prompt :

| Placeholder | Description | Exemple |
|------------|-------------|---------|
| `{TITLE}` | Titre de l'article | "Machine Learning for NLP" |
| `{AUTHORS}` | Auteurs | "Smith, J.; Doe, M." |
| `{DATE}` | Date de publication | "2024" |
| `{DOI}` | Digital Object Identifier | "10.1234/example" |
| `{URL}` | URL de l'article | "https://..." |
| `{ABSTRACT}` | Résumé de l'article | "This paper presents..." |
| `{TEXT}` | Texte complet extrait par OCR | "Full text..." (max 8000 caractères) |
| `{LANGUAGE}` | Langue cible | "français", "English", "español", etc. |

## ✏️ Exemples de Personnalisation

### Exemple 1 : Fiche minimaliste

```markdown
Tu es un assistant de recherche.

Résume cet article en 100 mots maximum en {LANGUAGE}.

**Article** : {TITLE}
**Auteurs** : {AUTHORS}

**Texte** :
{TEXT}

Format : HTML simplifié (<p>, <strong>)
```

### Exemple 2 : Fiche détaillée pour thèse

```markdown
Tu es un doctorant en [VOTRE DOMAINE].

Analyse cet article de manière critique pour ma thèse de doctorat.

## ARTICLE

**Titre** : {TITLE}
**Auteurs** : {AUTHORS}
**Date** : {DATE}
**Résumé** : {ABSTRACT}

## TEXTE COMPLET

{TEXT}

## CONSIGNE

Produis une fiche de lecture de 500 mots en {LANGUAGE} contenant :

1. **Contexte scientifique** : Positionnement dans la littérature
2. **Contribution originale** : Nouveauté de l'article
3. **Méthodologie détaillée** : Données, protocole, outils
4. **Résultats** : Findings principaux + chiffres clés
5. **Critique constructive** : Forces et faiblesses
6. **Pertinence pour ma thèse** : Comment utiliser cet article

Format : HTML avec `<h3>` pour les titres de section.
```

### Exemple 3 : Fiche orientée applications

```markdown
Tu es un data scientist.

Extrait les informations pratiques de cet article : {TITLE}

**Contexte** :
- Auteurs : {AUTHORS}
- Date : {DATE}
- DOI : {DOI}

**Contenu** :
{TEXT}

## CONSIGNE

Génère une fiche technique en {LANGUAGE} avec :

1. **Cas d'usage** : Applications concrètes
2. **Outils/Frameworks** : Technologies utilisées
3. **Code disponible** : Liens GitHub si mentionnés
4. **Datasets** : Jeux de données utilisés
5. **Résultats chiffrés** : Performances, métriques
6. **Reproductibilité** : Peut-on reproduire ?

200 mots max. Format HTML.
```

## 🎯 Bonnes Pratiques

### ✅ À faire

- **Structurer clairement** : Sections bien définies
- **Limiter la longueur** : Spécifier nombre de mots (le LLM respecte mieux)
- **Format HTML** : Toujours demander HTML simplifié
- **Langue explicite** : Utiliser `{LANGUAGE}` pour multilingue
- **Instructions précises** : "Liste 3 points clés" plutôt que "résume"

### ❌ À éviter

- **Prompts trop longs** : Le LLM peut perdre le fil
- **Instructions contradictoires** : "Sois bref" + "Détaille tout"
- **Placeholders inventés** : Seuls ceux listés ci-dessus fonctionnent
- **HTML complexe** : Éviter CSS, JavaScript, etc.

## 🔄 Workflow de Modification

1. **Éditer** : Modifiez [zotero_prompt.md](zotero_prompt.md)
2. **Sauvegarder** : Le fichier est rechargé automatiquement
3. **Tester** : Générez une fiche via l'interface
4. **Itérer** : Affinez selon les résultats

**Note** : Aucun redémarrage du serveur n'est nécessaire. Le fichier est relu à chaque génération.

## 🧪 Test Rapide

Pour tester votre prompt modifié :

1. Uploadez un export Zotero avec 1-2 articles
2. Traitez jusqu'à l'étape "Sparse Embeddings"
3. Cochez "Zotero Reading Notes"
4. Cliquez "Generate Zotero Notes"
5. Vérifiez le résultat dans Zotero

## 🆘 Dépannage

### Le prompt n'est pas pris en compte

- Vérifiez que le fichier est bien dans `app/utils/zotero_prompt.md`
- Vérifiez l'encodage : doit être UTF-8
- Consultez les logs : `logs/app.log` pour les erreurs

### Les placeholders ne sont pas remplacés

- Vérifiez l'orthographe exacte : `{TITLE}` (majuscules)
- Vérifiez les accolades : `{` et `}` (pas d'espaces)
- Exemple CORRECT : `{TITLE}`
- Exemple INCORRECT : `{ TITLE }`, `{{TITLE}}`, `[TITLE]`

### Fallback automatique

Si le fichier `zotero_prompt.md` est introuvable ou corrompu, le système utilise automatiquement un prompt de secours hardcodé. Un warning apparaîtra dans les logs.

## 📚 Ressources

- [Documentation OpenAI Prompting](https://platform.openai.com/docs/guides/prompt-engineering)
- [Anthropic Prompt Engineering](https://docs.anthropic.com/claude/docs/prompt-engineering)
- [Exemples de prompts académiques](https://github.com/anthropics/anthropic-cookbook)

## 💡 Exemples de Cas d'Usage

### Pour une revue de littérature

Ajoutez une section "Comparaison avec autres travaux" et demandez au LLM de citer les références mentionnées dans le texte.

### Pour un état de l'art

Demandez une classification thématique : "Quelle approche ? (théorique/empirique/méthodologique)"

### Pour une veille technologique

Focalisez sur : nouveautés, benchmarks, code open-source, implications pratiques.

---

**Astuce** : Gardez une copie de sauvegarde de votre prompt personnalisé avant de faire des modifications importantes !
