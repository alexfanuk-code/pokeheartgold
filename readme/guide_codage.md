# Protocole de Codage - Projet EverElectrum

Ce document définit les règles strictes pour maintenir l'intégrité du code et la stabilité des builds du projet EverElectrum (mod FR sur base pokeheartgold US).

---

## 0. Vue d'ensemble du projet

**Branche de travail :** `EverElectrum` (ne jamais travailler sur `master`)  
**Référence vanilla :** `upstream/master` (repo pret original)  
**Principe cardinal :** chaque zone de modification a son propre protocole. Ne pas mélanger les procédures.

### Zones de modification et leur nature

| Zone | Chemin | Trackée git | Outil |
|---|---|---|---|
| Textes | `files/msgdata/msg/*.gmm` | ✅ Oui | Éditeur texte |
| Graphismes NARC | `files/a/**/*.d/` | ✅ Oui (depuis fix gitignore) | nitroarc + scripts |
| Code source C/ASM | `src/`, `asm/`, `include/` | ✅ Oui | Éditeur + mwccarm |
| Artefacts build | `*.bin`, `*.h`, `*.narc`, `build/` | ❌ Non (gitignorés) | Générés par make |

---

## 1. Méthode de travail générale (cycle obligatoire)

```
MODIFIER → MAKE → TEST VISUEL CIBLÉ → COMMIT
```

**Règle stricte :** ne jamais committer sans que le make précédent ait réussi.  
**Règle stricte :** un commit = une modification logique cohérente (ex : "textes combat", "graphisme sac").

### Make rapide ciblé
```bash
# Textes seuls (rapide)
make files/msgdata/msg/msg_XXXX.bin

# Build ROM complet
make heartgold

# Nettoyage artefacts puis rebuild
make clean-msg && make heartgold
```

---

## 2. Protocole : Textes (GMM)

Les fichiers `.gmm` sont la **source de vérité** des textes. Le make reconstruit automatiquement les `.bin` (binaire chiffré) et `.h` (constantes C) depuis les GMM.

### Règles de modification GMM

- Ne modifier **que le contenu** entre `>` et `<` dans la balise `<language name="English">`.
- Ne jamais toucher aux attributs XML, aux `id`, aux `index`, aux balises `attribute`.
- Préserver toutes les balises de formatage internes : `{STRVAR_X}`, `\n`, `{COLOR}`, etc.

**Exemple correct :**
```xml
<language name="English">Texte français ici</language>
```
**À ne jamais faire :**
```xml
<language name="French">...</language>   <!-- casse le parseur msgenc -->
<language name="English" new_attr="x">   <!-- corrompt la structure -->
```

### Problème connu : backslash en fin de commentaire dans les .h

`msgenc` génère des `.h` où chaque message apparaît en commentaire `//` avant le `#define`. Si un texte GMM se termine par `\` (backslash), le préprocesseur C interprète cela comme une continuation de ligne et **avale le `#define` suivant**.

**Symptôme :** erreur de compilation sur un symbole non défini (`msg_XXXX_YYYYY`).  
**Vérification après make :**
```bash
grep -n "\\$" files/msgdata/msg/msg_XXXX.h
```
**Correction :** dans le GMM correspondant, supprimer tout `\` terminal dans les textes.

---

## 3. Protocole : Graphismes NARC (dossiers .d/)

Les NARCs du jeu sont stockés sous forme binaire dans `files/a/X/Y/Z` (trackés). Le dossier `files/a/X/Y/Z.d/` contient le même NARC dépaquété (fichiers numérotés 5 chiffres) — maintenant tracké git.

### Workflow complet graphismes FR

```
1. Extraire les NARCs vanilla vers .d/
   → nitroarc -xf files/a/X/Y/Z -C files/a/X/Y/Z.d/

2. Injecter les graphismes FR dans les .d/
   → python fr_extract/auto_graphic_swap.py

3. Corriger le build system (DIFF_ARCS vs non-DIFF_ARCS)
   → python fr_extract/fix_fr_injection.py

4. Rebuild
   → make heartgold
```

**Rappel architecture Makefile :**
- **Non-DIFF_ARCS** : le binaire `files/a/X/Y/Z` est utilisé directement. `fix_fr_injection.py` le repack depuis le `.d/`.
- **DIFF_ARCS** (68 entrées dans `filesystem.mk`) : `make` reconstruit depuis `files/data/namein/` → injecter les NCGR/NCLR FR dans les fichiers source US.

`fix_fr_injection.py` est **idempotent** : peut être relancé sans risque après tout reset.

---

## 4. Protocole : Code Source C/ASM

### Formatage automatique (obligatoire avant commit)
```bash
./format.sh          # formate tout le projet
```
Le projet utilise `clang-format` v18+. **Exception :** les blocs assembleur doivent être protégés.

### Protection des blocs ASM (CRITIQUE)
```c
// clang-format off
asm void MaFonctionAsm() {
    push {lr}
    pop {pc}
}
// clang-format on
```

### Fonctions NONMATCHING
Pour tester du code C sans perdre la référence binaire originale :
```c
#ifdef NONMATCHING
void MaFonction() {
    // Nouveau code C pour le mod
}
#else
// clang-format off
asm void MaFonction() {
    // Code ASM original (binaire identique requis)
}
// clang-format on
#endif // NONMATCHING
```

---

## 5. Configuration Git (setup initial)

À exécuter une fois à la racine du repo :
```bash
git config --local core.hooksPath .githooks/
git config alias.clang-format clang-format
```

### Gitignore — règles importantes
- `*.d` global → ignoré (depfiles compilateur), **sauf** `files/a/**/*.d/` (exception explicite)
- `*.NCLR` dans `files/` → ignoré, **sauf** `files/a/**/*.d/*.NCLR` (sources graphiques)
- Ne jamais committer les artefacts build : `.bin`, `.h`, `.narc`, `build/`

### Branches
- `EverElectrum` = branche de travail principale
- Une sous-branche par fonctionnalité si développement parallèle (ex: `feat-textes-combat`)
- Toujours vérifier `git diff upstream/master -- [zone]` avant de committer pour s'assurer de la portée du changement

---

## 6. Tests visuels ciblés

Après chaque make réussi, tester **uniquement la zone modifiée** dans l'émulateur :
- **Textes** : naviguer dans les menus concernés, déclencher les dialogues modifiés
- **Graphismes** : vérifier l'affichage dans le contexte exact (combat, sac, pokégear, etc.)
- **Code** : tester le chemin exact touché par la modification

Ne pas faire de tests exhaustifs à chaque étape : ils sont chronophages et masquent les régressions locales. Faire un test complet uniquement après un jalon majeur.
