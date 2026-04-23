# Protocole de Codage - Projet EverElectrum

Ce document définit les règles strictes pour maintenir l'intégrité du code et la compatibilité avec le compilateur mwccarm.

## 1. Formatage Automatique
Le projet utilise impérativement \`clang-format\` (v18+). 
- \*\*Règle :\*\* Ne jamais soumettre de code non formaté.
- \*\*Script global :\*\* Exécuter \`./format.sh\` pour traiter l'ensemble de l'arborescence.
- \*\*Exception :\*\* L'assembleur (ASM).

## 2. Protection de l'Assembleur (CRITIQUE)
Le formateur ne reconnaît pas la syntaxe inline ASM de \`mwccarm\`. Tout bloc assembleur doit être protégé par des directives de commentaire :

```c
// clang-format off
asm void MaFonctionAsm() {
    push {lr}
    // ...
    pop {pc}
}
// clang-format on
```

## 3\. Gestion des Fonctions Non Identiques (NONMATCHING)

Pour tester des modifications en C sans perdre la référence originale ou pour les fonctions ne produisant pas encore un binaire identique :

```C
#ifdef NONMATCHING
void MaFonction() {
    // Ton nouveau code C pour le mod
}
#else
// clang-format off
asm void MaFonction() {
    // Code original nécessaire pour la compilation de base
}
// clang-format on
#endif // NONMATCHING
```

## 4. Workflow et Configuration Git

Pour que l'environnement soit opérationnel, ces commandes doivent être exécutées à la racine :

-   **Activation des hooks :** `git config --local core.hooksPath .githooks/`.
-   **Alias de formatage :** `git config alias.clang-format clang-format` (Essentiel pour le fonctionnement du hook).
-   **Branches :** Créer une branche par fonctionnalité (ex: `feat-stats-germignon`).
-   **Validation :** Vérifier la compilation (`make`) impérativement AVANT chaque commit.