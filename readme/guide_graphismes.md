# Guide Graphismes FR — Projet EverElectrum

Ce document couvre l'architecture graphique du projet, le bug de double-compression découvert et résolu, le workflow d'injection FR, et le protocole détaillé pour modifier l'écran titre.

---

## 1. Architecture : DIFF_ARCS vs non-DIFF_ARCS

Les graphismes du jeu sont stockés dans des NARCs (Nintendo ARChive) sous `files/a/X/Y/Z`.

### Non-DIFF_ARCS
Le binaire `files/a/X/Y/Z` est utilisé **directement** par makerom. Make ne le reconstruit pas — il est tracké git et copié tel quel dans la ROM.

**Injection FR :** remplacer le fichier binaire par l'équivalent FR packé. Aucun risque de re-compression.

### DIFF_ARCS (68 entrées dans `filesystem.mk`)
Make **reconstruit** le NARC depuis un dossier source (ex : `files/demo/title/titledemo/`) ou un NARC source (ex : `files/system/touch_subwindow.narc`), puis le copie vers `files/a/X/Y/Z`.

**Règle critique :** les fichiers sources dans ces dossiers doivent être **non-compressés**. Make applique lui-même la compression LZ77 (règle `%.lz: %`) pour les slots qui en ont besoin.

---

## 2. Bug résolu : double-compression LZ77

### Symptôme
ROM démarrée en écran blanc persistant (son OK → moteur ARM7 fonctionnel, seul l'affichage crashait).

### Cause
L'injection FR plaçait des fichiers **déjà compressés LZ77** (extraits bruts des NARCs FR via `nitroarc -xf`) dans les dossiers sources des DIFF_ARCs. Make les re-compressait → slots doublement compressés → NARC corrompu → crash au chargement graphique.

### Identification
```bash
# Fichier LZ77 : commence par 0x10
python3 -c "import sys; d=open('fichier.NCGR','rb').read(4); print(hex(d[0]))"
# 0x10 → compressé LZ77  |  0x52 (= 'R', début de RGCN) → brut NCGR valide
```

### Fix appliqué
`fr_extract/inject_fr_graphics.py` — ajout d'une décompression LZ77 (type `0x10`) avant écriture dans chaque dossier source DIFF_ARC (Phase 2a). Les Phases 1 (non-DIFF_ARCs) et 2b (NARCs binaires statiques) ne sont pas affectées car les fichiers y sont utilisés tels quels sans re-compression.

---

## 3. Workflow d'injection FR (état stable)

### Commandes
```bash
# Depuis C:/compil/pokeheartgold/
python3 /C/compil/fr_extract/inject_fr_graphics.py
make heartgold -j$(nproc)
```

Le script est **idempotent** : peut être relancé à tout moment sans risque, même après un `make clean` ou un `git checkout master -- files/`.

### Restauration vanilla + re-injection (reset complet)
```bash
# 1. Restaurer tous les fichiers graphiques depuis master (vanilla propre)
git diff --name-only master HEAD -- files/ | grep -v "^files/msgdata/" | grep -v "^files/.gitignore" | xargs git checkout master --

# 2. Ré-injecter les graphismes FR
python3 /C/compil/fr_extract/inject_fr_graphics.py

# 3. Rebuild
make heartgold -j$(nproc)
```

### Ce que fait le script
| Phase | Quoi | Comment |
|-------|------|---------|
| 1 | Non-DIFF_ARCs (200 NARCs) | Pack `data_fr/a/X/Y/Z.d/` → `files/a/X/Y/Z` via nitroarc |
| 2a | DIFF_ARCs dossier source (33 NARCs) | Copie fichiers FR dans dossier US + **décompression LZ77** |
| 2b | DIFF_ARCs NARC binaire (29 NARCs) | Pack `data_fr/a/X/Y/Z.d/` → NARC source |

---

## 4. Modification de l'écran titre

### Localisation
```
DIFF_ARC : files/demo/title/titledemo/  →  files/a/0/4/6
Make file : files/demo/title/titledemo.mk
Code C    : src/title_screen.c
```

### Particularité importante : pas de LZ77 dans titledemo

Contrairement à `gs_opening.narc`, **aucun slot de titledemo n'est compressé LZ77**. Les fichiers sources dans `files/demo/title/titledemo/` sont tous bruts. Make les pack directement sans étape `.lz`.

### Cartographie des slots (HeartGold)

| Slot | Type | Couche | Rôle |
|------|------|--------|------|
| 00000000 | NSCR | SUB_BG LYR_2 | Tilemap fond commun HG+SS |
| 00000001 | NCGR | SUB_BG LYR_2 | Tuiles fond (SS uniquement) |
| 00000002 | NCLR | — | Palette fond (SS) + palette ext SUB_BG |
| 00000003 | NCGR | SUB_BG LYR_2 | Tuiles fond (**HG**) |
| 00000004 | NCLR | — | Palette fond (**HG**) + palette ext SUB_BG |
| 00000013 | NCLR | MAIN_BG | Palette fond MAIN (**HG**) |
| 00000014 | NCLR | MAIN_BG | Palette fond MAIN (SS) |
| 00000015 | NCGR | SUB_BG LYR_1 | Tuiles couche logo (4bpp, HG+SS) |
| 00000017 | NSCR | SUB_BG LYR_1 | Tilemap couche logo (HG+SS) |
| 00000034 | NCGR | SUB_BG LYR_3 | Tuiles couche titre texte (**HG**, 8bpp) |
| 00000035 | NSCR | SUB_BG LYR_3 | Tilemap couche titre texte (**HG**) |
| 00000036 | NCGR | SUB_BG LYR_3 | Tuiles couche titre texte (SS, 8bpp) |
| 00000037 | NSCR | SUB_BG LYR_3 | Tilemap couche titre texte (SS) |
| 00000020–24 | NSB* | 3D MAIN | Modèle Lugia + animations (SS) |
| 00000025–29 | NSB* | 3D MAIN | Modèle Ho-Oh + animations (**HG**) |
| 00000038–40 | NSB* | 3D MAIN | Étincelles / effets (**HG**) |
| 00000041–43 | NSB* | 3D MAIN | Étincelles / effets (SS) |

Les slots **HG** prioritaires pour la localisation FR : `00000003`, `00000004`, `00000013`, `00000034`, `00000035`.

### Fichiers éditables via PNG (workflow make natif)

`titledemo.mk` déclare des sources PNG pour certains NCGR. Éditer le PNG → make régénère l'NCGR automatiquement :

```
titledemo_00000001.png → titledemo_00000001.NCGR  (8bpp, SS)
titledemo_00000003.png → titledemo_00000003.NCGR  (8bpp, HG)
titledemo_00000034.png → titledemo_00000034.NCGR  (8bpp, HG logo bas)
titledemo_00000036.png → titledemo_00000036.NCGR  (8bpp, SS logo bas)
titledemo_00000015.png → titledemo_00000015.NCGR  (4bpp, logo commun)
```

Règles make :
- `00000001`, `00000003`, `00000034`, `00000036` : `-version101 -sopc -bitdepth 8`
- `00000015` : `-version101 -sopc` (4bpp)

### Workflow modification écran titre

#### Option A — Modifier via PNG (NCGR avec source PNG)
```bash
# 1. Éditer le PNG dans files/demo/title/titledemo/
#    (GIMP, Aseprite, etc. — respecter la palette et le nombre de couleurs)

# 2. Supprimer l'ancien NCGR pour forcer la regen
rm files/demo/title/titledemo/titledemo_XXXXX.NCGR

# 3. Rebuild ciblé
make files/a/0/4/6

# 4. Test puis rebuild complet si ok
make heartgold -j$(nproc)
```

#### Option B — Modifier NCGR/NSCR/NCLR directement (tous slots)
```bash
# 1. Éditer le fichier avec NitroPaint (outil externe recommandé)
#    - Ouvrir titledemo_XXXXX.NCGR / .NSCR / .NCLR
#    - Modifier, sauvegarder en place

# 2. Rebuild ciblé
make files/a/0/4/6

# 3. Test puis rebuild complet si ok
make heartgold -j$(nproc)
```

### Make ciblé (rebuild rapide)
```bash
# Rebuild uniquement le NARC titledemo → files/a/0/4/6 (quelques secondes)
make files/a/0/4/6

# Rebuild ROM complète
make heartgold -j$(nproc)
```

**Note :** `make files/a/0/4/6` seul ne produit pas de ROM. Il faut quand même `make heartgold` pour embedder le NARC dans la ROM finale. Le make incrémental ne recompile que ce qui a changé — rapide.

### Garder les graphismes FR après modification

Après avoir modifié un fichier dans `files/demo/title/titledemo/`, le script `inject_fr_graphics.py` **écraserait** votre modification si relancé. Deux options :

1. **Committer le fichier modifié avant** toute re-injection → il sera tracké git et protégé
2. **Exclure ce dossier** du script en ajoutant une entrée dans `EXCLUDED_SOURCES` dans `inject_fr_graphics.py` si vous ne voulez jamais écraser ce NARC avec la version FR automatique

---

## 5. Autres NARCs fréquemment modifiés

| NARC | Chemin source | files/a/ | Contenu |
|------|---------------|----------|---------|
| gs_opening | `files/demo/opening/gs_opening/` | `2/6/2` | Cinématique intro boot (LZ77 requis) |
| guinness | `files/application/guinness/` | `2/6/0` | Écran Wi-Fi Guinness |
| namein | `files/data/namein/` | `0/3/1` | Interface saisie nom |
| plist_gra | `files/graphic/plist_gra/` | `0/2/1` | Graphismes liste Pokémon |
| zukan_gra | `files/graphic/zukan_gra/` | `0/6/8` | Graphismes Pokédex |
| font | `files/graphic/font/` | `0/1/6` | Police (pre-built, ne pas reconstruire) |

**Attention gs_opening :** contrairement à titledemo, les NCGR/NSCR de gs_opening **sont compressés LZ77** dans le NARC (règle `%.lz: %` dans le build). Les sources dans `files/demo/opening/gs_opening/` doivent être brutes (non-LZ77). Make compresse automatiquement.

---

## 6. Règles de sécurité pour les graphismes

1. **Toujours vérifier qu'un fichier source DIFF_ARC est brut** avant de le committer :
   ```bash
   python3 -c "import sys; d=open('fichier.NCGR','rb').read(1); print('LZ77!' if d[0]==0x10 else 'OK brut')"
   ```

2. **Ne jamais committer les artefacts build** : `.lz`, `.narc`, fichiers dans `build/`

3. **Test obligatoire** : après chaque modification graphique, `make heartgold` + test émulateur sur la zone concernée

4. **Source de vérité vanilla** : `master` branch — utiliser `git show master:chemin/fichier` pour récupérer n'importe quel fichier vanilla
