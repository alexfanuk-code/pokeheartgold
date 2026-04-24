# EverElectrum — Workflow de réimplémentation

## Sources de vérité

| Source | Ce qu'elle valide |
|--------|-------------------|
| `C:\compil\pokeheartgold` branch `master` | La version vanilla - unique point de comparatif en cas de casse |
| `C:\compil\pokeheartgold` branch `EverElectrum` | L'état sain actuel du mod - ne pas comparer les commits entre eux en cas de casse : role de la branch master|
| `C:\pokeheartgold\readfirst\workflow-technique.md` et `C:\pokeheartgold\readfirst\build_manifest.md` (ancien repo) | Uniquement le "quoi" : ce qu'on a cherché à ajouter/modifier dans le jeu, pas le comment, pas le avec quoi (mod cassé) |

**Tout ce qui n'est pas vérifiable directement dans l'un de ces trois endroits est à considérer comme non fiable jusqu'à vérification (mécaniques, techniques, scripts, outils, flags...).**

---

## Principe général

```
LIRE l'ancien repo (intent)
→ VÉRIFIER dans le repo sain (réalité)
→ IMPLÉMENTER
→ MAKE
→ TEST sur partie neuve
→ COMMIT atomique
```

Jamais de copier-coller depuis l'ancien repo. Jamais de commit sans make réussi.

---

## Ordre d'implémentation — raisonnement

L'ordre est dicté par les dépendances réelle. Chaque tâche est à faire indépendamment, dans l'ordre strict de la checklist des tâches (en fin de document).

```
Couche 1 : Données et constantes
           (tout le reste en dépend)
     ↓
Couche 2 : Code C — systèmes centraux
           (les scripts en dépendent)
     ↓
Couche 3 : Données de jeu (tables JSON / CSV)
           (équilibrage, évolutions, rencontres)
     ↓
Couche 4 : Scripts et événements
           (contenu, quêtes, PNJ)
     ↓
Couche 5 : Graphismes
           (isolé, pipeline séparé)
     ↓
Couche 6 : Overlays et hooks sensibles
           (en dernier — risque de régression le plus élevé)
```

---

## Couche 1 — Données et constantes

Tout commit de code C ou de script qui référence un flag, une var, un item ou un bloc save doit avoir ses constantes déclarées **avant**. C'est la première chose à faire.

### Étape 1.1 — Extension des tableaux de flags et de variables

**Vérification préalable dans le repo sain :**
Lire `include/constants/flags.h` et `include/constants/vars.h` sur `EverElectrum` pour connaître l'état actuel.

**Référence dans l'ancien repo :**
Lire les mêmes fichiers sur l'ancien repo pour identifier les valeurs ajoutées (plages custom, nouvelle taille de tableau).

**Implémenter :**
Reporter les mêmes valeurs dans le repo sain. Vérifier l'absence de collision avec les constantes vanilla existantes.

**Vérification :** `make` propre. **Commit.**

### Étape 1.2 — Déclaration des items custom

**Vérification préalable :**
Lire `include/constants/items.h` sur `EverElectrum` pour connaître l'état actuel.

**Référence dans l'ancien repo :**
Identifier les items ajoutés (plage d'indices, noms).

**Implémenter :**
Ajouter les déclarations dans `include/constants/items.h` du repo sain. Vérifier que `ITEMS_COUNT` est cohérent.

**Vérification :** `make` propre. **Commit.**

### Étape 1.3 — Bloc de sauvegarde custom

**Vérification préalable :**
Lire `src/save_arrays.c` et l'include correspondant sur `EverElectrum`. Identifier si le bloc existe déjà.

**Référence dans l'ancien repo :**
Lire le header et le `.c` du bloc save custom. Comprendre la struct, sa taille, ses accesseurs, son ID d'enregistrement.

**Implémenter :**
Créer ou compléter le header et le `.c` dans le repo sain. Enregistrer l'ID dans `save_arrays.c`.

**Vérification critique :** La taille de la struct doit être vérifiable statiquement. `make` propre. Tester que le jeu démarre et sauvegarde sans crash. **Commit.**

---

## Couche 2 — Code C central

### Étape 2.1 — Équilibrage global (formules de jeu)

Ces modifications touchent des fonctions isolées sans dépendances externes autres que les constantes vanilla. Elles sont donc sûres à faire tôt.

**Pour chaque modification d'équilibrage (EXP, argent, loot, Masuda…) :**

1. Identifier dans l'ancien repo la fonction modifiée et la nature exacte du patch
2. Localiser la même fonction dans le repo sain (peut avoir le même nom ou être NONMATCHING)
3. Appliquer le patch de façon minimale et ciblée
4. `make` → test isolé de la fonctionnalité → **commit séparé par fonctionnalité**

### Étape 2.2 — Handlers d'items custom

Dépend de : Étape 1.2 (items déclarés) et Étape 1.3 (bloc save accessible).

**Pour chaque handler :**

1. Lire dans l'ancien repo la logique du handler (fonction menu + fonction check)
2. Identifier comment il lit/écrit dans le bloc save custom
3. Implémenter dans `field_use_item.c` du repo sain
4. Enregistrer le handler dans la table des field-use (vérifier l'index)
5. `make` → test : utiliser l'item en jeu, vérifier le comportement → **commit par handler ou groupe cohérent**

### Étape 2.3 — Système de capture légendaire (hook central)

Ce système conditionne toutes les quêtes du Lot E. Il doit exister et fonctionner avant le premier script de quête.

1. Lire dans l'ancien repo les fonctions concernées dans `encounter.c`, `scrcmd_c.c`, `start_menu.c` et l'overlay PC
2. Comprendre la logique complète : détection de l'espèce légendaire à la capture, envoi au PC, verrouillage
3. Implémenter dans le repo sain fichier par fichier
4. `make` → test : capturer un légendaire concerné, vérifier qu'il va au PC et ne peut pas être retiré avant la condition finale → **commit**

### Étape 2.4 — Commandes script custom (`scrcmd_c.c`)

Toute commande custom appelée depuis les scripts doit être enregistrée dans la table avant les scripts qui l'utilisent.

1. Lire dans l'ancien repo toutes les fonctions `ScrCmd_*` custom ajoutées
2. Les implémenter dans le repo sain
3. Les enregistrer dans la table de dispatch
4. `make` → test minimal d'une commande → **commit**

---

## Couche 3 — Données de jeu (tables JSON / CSV)

Dépend de : Couche 1 (constantes), car certaines tables référencent des indices.

### Étape 3.1 — Table des items (`item_data.csv`)

1. Comparer l'état de `files/itemtool/itemdata/item_data.csv` entre l'ancien repo et le repo sain
2. Ajouter uniquement les lignes correspondant aux items déclarés en Étape 1.2
3. `make` → vérifier que le NARC item est correctement rebuilté → **commit**

### Étape 3.2 — Évolutions (`evo.json`)

1. Identifier dans l'ancien repo les entrées modifiées (évolutions remplaçant les échanges, nouvelles évolutions)
2. Reporter uniquement ces entrées dans le repo sain — ne pas remplacer le fichier entier
3. `make` → test d'une évolution modifiée → **commit**

### Étape 3.3 — Tables de rencontres

1. Identifier dans l'ancien repo les modifications apportées aux tables HG/SS
2. Reporter dans le repo sain de façon chirurgicale
3. `make` → test de rencontres sur une zone modifiée → **commit**

---

## Couche 4 — Scripts et événements

Dépend de : Couches 1, 2 et 3 entièrement validées.

### Principe pour chaque script

1. Identifier dans l'ancien repo le fichier script concerné et sa logique
2. Vérifier dans le repo sain si le fichier existe déjà et dans quel état
3. Vérifier que tous les flags, vars, items et commandes script référencés sont bien déclarés (Couches 1 et 2)
4. Implémenter ou compléter le script
5. `make` → test du scénario exact couvert par le script → **commit**

### Ordre interne des scripts

Partir des plus simples vers les plus complexes, selon cette logique :

**D'abord** les scripts globaux (hook présent partout) — ils conditionnent les autres.

**Ensuite** les scripts QoL sans chaîne d'événements (CTs, HMs, Safari, essaims, vitesse) — peu de dépendances, risque faible.

**Ensuite** les scripts de distribution d'objets simples (un PNJ donne un item, pose un flag) — testables isolément.

**Ensuite** les scripts de distribution de Pokémon (starters supplémentaires) — mêmes conditions que ci-dessus mais avec gestion de l'espèce et de la var associée.

**Ensuite** les quêtes légendaires, du plus simple au plus complexe :
- Spawn direct sur usage d'item (une seule map, une seule étape)
- Chaînes à deux étapes (item → PNJ → spawn)
- Chaînes multi-maps ou multi-conditions
- Chaînes impliquant plusieurs légendaires liés entre eux

**Enfin** les scripts de Pokégear et les hooks dans les overlays (cf. Couche 6).

### Vérification systématique pour chaque quête/événement

- [ ] Tous les flags utilisés sont déclarés et dans la bonne plage
- [ ] Toutes les vars utilisées sont déclarées
- [ ] Tous les items référencés existent dans `item_data.csv`
- [ ] Toutes les commandes script custom utilisées sont enregistrées
- [ ] Le scénario fonctionne de bout en bout sur partie neuve
- [ ] La progression persiste après save/load
- [ ] Les fonctionnalités des commits précédents ne régressent pas

---

## Couche 5 — Graphismes

À traiter en **lot isolé**, séparé des autres couches. Ne jamais mélanger un commit graphique avec un commit script ou C.

### Règle préalable : DIFF_ARC vs non-DIFF_ARC

Avant toute modification graphique, identifier si le NARC cible est rebuilté depuis des sources (DIFF_ARC) ou traité comme binaire direct (non-DIFF_ARC). Cette information est dans `filesystem.mk` du repo sain. Le traitement est différent selon le cas.

### Étape 5.1 — Vérifier les assets FR déjà injectés

Confirmer dans `EverElectrum` que les graphismes FR vanilla sont bien versionnés et que le build les inclut correctement.

### Étape 5.2 — Assets custom (écran titre, etc.)

Pour chaque asset graphique custom identifié dans l'ancien repo :

1. Récupérer le fichier depuis l'ancien repo
2. **Vérifier sa cohérence** : taille, format, absence de corruption apparente — ne pas faire confiance aveuglément
3. Si cohérent : l'intégrer dans le repo sain via le pipeline documenté dans `guide_graphismes.md`
4. Si DIFF_ARC : modifier les sources, laisser le pipeline rebuilder le NARC
5. Si non-DIFF_ARC : remplacer le binaire, versionner
6. `make` → test visuel → **commit**

### Vérification critique : `touch_subwindow.narc`

Avant tout make impliquant des graphismes, vérifier la taille de ce fichier contre la taille vanilla connue du repo sain. S'il est corrompu, le restaurer depuis `master` avant de continuer.

---

## Couche 6 — Overlays et hooks sensibles

En dernier. Ces fichiers compilent séparément, les erreurs peuvent être silencieuses, et les régressions sont difficiles à isoler.

### Pour chaque hook overlay

1. Lire dans l'ancien repo la modification exacte (fonction ajoutée, point d'injection)
2. Comprendre pourquoi ce hook est dans un overlay et non dans le code principal
3. Implémenter de façon chirurgicale dans le repo sain
4. `make` → tester le chemin de code exact couvert par le hook → **commit**

### Vérification supplémentaire

Après chaque hook overlay, re-tester au moins une fonctionnalité des couches précédentes pour détecter toute régression silencieuse.

---

## Checklist de commit (universelle)

Avant tout commit :

- [ ] Test terrain sur partie neuve réalisé
- [ ] Fonctionnalités précédentes non régressées
- [ ] Un seul sujet logique dans le commit
- [ ] Message clair : `[CoucheX] Description concise`
- [ ] Aucun artefact de build inclus dans le commit

---

# EverElectrum - Checklist des tâches - à mettre à jour avant chaque commit

## Couche 1 — Données et constantes

- flags.h / vars.h : extension des plages EE (prérequis absolu à tout le reste)
- SaveEverElectrum : struct + enregistrement ID 41 dans save_arrays (C-4 en dépend)
- items.h : déclaration items 536–545, ITEMS_COUNT = 545 (C-5)
- item_data.csv : entrées items 536–544 (C-5)


## Couche 2 — Code C central

- Nerf EXP −50 % — battle_command.c BtlCmd_CalcExpGain
- Argent ×1,5 — battle_command.c CalcPrizeMoney
-Pépite 10 % wild — pokemon.c WildMonSetRandomHeldItem
- Masuda par Trainer ID — daycare.c Save_Daycare_MasudaCheck
- TMs illimitées — party_menu_items.c PartyMenu_LearnMoveToSlot
- HMs passives — scrcmd_party.c + field_move.c
- HMs effaçables — item.c MoveIsHM → toujours FALSE
- Safari sans compteur — safari_zone.c sub_0202F798
- C-1 Sceau des Légendes : hook post-capture → envoi PC — encounter.c
- C-2 Sceau : retrait PC bloqué avant Red — scrcmd_c.c + start_menu.c
- C-3 Fateful Encounter helpers Célébi / Arceus — pokemon.c
- C-4 Pierre Chromatique : bits SaveEverElectrum + encounter_check.c + handler field-use
- C-5 field_use_item.c : handlers d'usage indices 30–36
- Essaims — partie C : field_warp_tasks.c hook map + scrcmd_strbuf.c ScrCmd_EverElectrum_PrepareSwarmNotif
- Commandes script custom : ScrCmd_SetCelebiFatefulEncounter (cmd 855) + ScrCmd_SetArceusFatefulEncounter (cmd 856) — scrcmd_c.c (les -scripts de C-4 en dépendent)


## Couche 3 — Tables de données jeu

- evo.json : évolutions sans échange + Gen 4 manquantes
- gs_enc_data.json : fusion tables rencontres HG/SS + placements exclusifs


## Couche 4 — Scripts et événements

- Bank EVERYWHERE : initialiser le script global (Darkrai, Regice, Phione s'y accrochent)
- S-1 Sac de Couchage — distribution Maman, Bourg Geon
- S-2 Laptop Sylphe — distribution scientifique, Doublonville
- S-3 Poké-Analyseur — distribution assistant Orme, Bourg Geon
- S-4 Herboriste Doublonville — ChangeMonNature
- S-5 Shiny Charm — récompense Chen / Pokédex National
- S-6 Concentrateur Oméga — Chen, 16 badges + Red
- Essaims — partie script : scr_seq_0003_072 + msg_0040
- ST-1 Starters Johto restants — Prof. Orme
- ST-2 Starters Kanto restants — Prof. Chen
- ST-3 Starters Hoenn restants — Pierre Rochard
- ST-4 Starters Sinnoh ×3 — Cynthia / Ruines Sinjoh
- L-Mew — Vieille Carte, Route 25
- L-Célébi / GS Ball — Chen → Fargas → Autel Ilex
- L-Arceus / Flûte Azur — chercheur D24R0102 → bg_event D24R0101 (à faire avant toutes les quêtes gatant sur FLAG_EE_ARCEUS_APPARU)
- L-Motisma — sous-sol Tour Radio
- L-Jirachi / Étoile Filante — astronome Mont Lune, gate Red
- L-Deoxys / Carnet de Recherche — chercheur Mont Lune, gate Red + Mewtwo
- L-Latios / Mystécristal — Fan Club Carmin → Musée Argenta, gate Latias
- L-Groudon / Orbe Rouge — M. Pokémon R30R0101, gate Kyogre
- L-Gardiens du Lac / Notes de Cynthia — Sinjoh D51R0301, gate Arceus apparu
- L-Heatran / Clé Thermaïque — géologue T31PC0101, gate Groudon + Kyogre
- L-Manaphy + Phione / Invitation — port Carmin, gate Lugia + Kyogre
- L-Darkrai + Cresselia / Bandeau Ombral — médecin Acajou, gate Lugia + trio Kanto
- L-Shaymin / Gracidée — fleuriste Doublonville + Parc national, gate Red + Pokédex Johto
- L-Regirock / Regice / Registeel — runes via D24R0102, gate Arceus apparu + Sinjoh
- L-Regigigas — Tour Éboulis B1F, gate trio Regis capturés
- Fusion scénarios Kimono HG/SS — Tour Radio + Rosalia + Tour Carillon + Tourb'Îles
- Théon l'Arpenteur — oracle quêtes (en dernier : tous les flags EE doivent exister)


## Couche 5 — Graphismes

- Injection écran titre DS — titledemo NCGR/NSCR/NCLR + commentaire title_screen.c


## Couche 6 — Overlays et hooks sensibles

- Rematchs Pokématos — overlay_2_gear_phone.c + overlay_101_021F1D74.c + phone_scripts_generic.c + encounter.c + msg_0271