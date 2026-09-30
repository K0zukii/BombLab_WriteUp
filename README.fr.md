# CMU Binary Bomb Lab (x86-64) - Write-Up complet

## Contexte du projet
Ce dépôt documente la résolution complète du célèbre **Binary Bomb Lab** (créé par l'Université Carnegie Mellon), réalisé dans le cadre du cours d'architecture et d'assembleur x86-64 dispensé par **OST2 (OpenSecurityTraining2)**.

**Point de départ :** Avant de démarrer ce laboratoire, je n'avais absolument aucune expérience préalable en Rétro-Ingénierie (Reverse Engineering). Ce projet s'est étalé sur plusieurs jours, avec des pauses nécessaires pour assimiler les concepts théoriques. La courbe d'apprentissage a été particulièrement abrupte, notamment sur les phases 5 et 6 qui nécessitent une forte compréhension des pointeurs et des structures de données en C. Cependant, la persévérance et le croisement des outils d'analyse dynamique et statique m'ont permis d'aller jusqu'au bout.

**Outils utilisés :** WinDbg (Analyse dynamique), Ghidra (Analyse statique / Décompilation), Outils Sysinternals (`strings`).

---

## Phase 1 : Reconnaissance statique et comparaison de chaînes

### 1. Reconnaissance initiale (Strings)
Avant même de désassembler le binaire, j'ai effectué une première phase de reconnaissance avec l'outil `strings` de Sysinternals pour extraire le texte en clair du fichier `bomb.exe`.
Parmi les résultats, plusieurs chaînes suspectes ressemblaient à des mots de passe potentiels, notamment :
* `passphrase`
* `DrEvil`
* `I am just a renegade hockey mom.`

![Résultat de la commande Strings](Screenshots/StringsCommand_Terminal.png)

### 2. Analyse Dynamique (WinDbg)
Pour confirmer quel mot de passe était le bon, j'ai choisi d'analyser la fonction `phase_1` dynamiquement sous WinDbg en lisant directement l'assembleur.
En exécutant le code pas à pas (`p`), j'ai repéré la préparation des arguments juste avant l'appel à la fonction de vérification (`strings_not_equal`). Le programme chargeait une adresse mémoire spécifique dans un registre.

Pour inspecter le contenu de cette adresse, j'ai utilisé la commande d'affichage de chaîne ASCII dans WinDbg (`da [adresse]`). La mémoire a révélé la chaîne exacte attendue par le programme.

![Inspection de la mémoire sous WinDbg](Screenshots/Phase1_windbg.png)

### 3. Solution
Le mot de passe pour désamorcer la phase 1 est : `I am just a renegade hockey mom.`

---

## Phase 2 : Boucles et structures de données (Tableaux)

### 1. Analyse Dynamique et Statique
L'analyse de la fonction `read_six_numbers` appelée au début de la phase 2 a révélé l'utilisation de `sscanf` avec un format nécessitant 6 entiers. En observant les décalages de 4 octets (`+4`, `+8`, etc.) sur la pile dans WinDbg, j'ai déduit que l'entrée était stockée dans un tableau de 6 entiers.

![Format sscanf trouvé dans WinDbg](Screenshots/Phase2_format_found_WinDBG.png)

En passant sur Ghidra pour analyser la logique de validation, j'ai identifié deux conditions critiques :
* Le premier entier du tableau doit être égal à `1`.
* Une boucle itère sur les 5 entiers restants avec la logique suivante : `array[i] = array[i-1] * 2`.

![Boucle de validation dans Ghidra](Screenshots/Phase2_Function_Ghidra.png)

### 2. Résolution
Chaque nombre doit être le double du précédent. En partant de 1, la suite logique est une progression géométrique de raison 2.
Le mot de passe de la phase 2 est : `1 2 4 8 16 32`

---

## Phase 3 : Flux de contrôle et Fall-through Switch

### 1. Reconnaissance des arguments (WinDbg)
J'ai d'abord exécuté la phase 3 dans WinDbg. En inspectant les arguments passés à `sscanf` avec la commande `da`, j'ai identifié la chaîne de format `%d %d`, indiquant que la bombe attendait deux nombres entiers.

![Arguments sscanf sous WinDbg](Screenshots/Phase3_format_found_WinDBG.png)

### 2. Analyse de l'algorithme (Ghidra)
Le décompilateur a révélé une structure de type `switch` basée sur le premier entier fourni. L'analyse a mis en évidence deux contraintes majeures :
* Une vérification finale exige que le premier entier soit inférieur ou égal à 5. Les cas 6 et 7 sont donc des impasses.
* Le bloc `switch` ne contient aucun `break` avant le cas 7 (comportement de *fall-through*). Cela signifie que le programme exécute en cascade toutes les opérations mathématiques des cas inférieurs.

![Structure Switch dans Ghidra](Screenshots/Phase3_Function_Ghidra.png)

### 3. Résolution
J'ai choisi d'utiliser le cas `5`. Le premier argument est donc `5`.
La variable de validation démarre à 0 et subit les opérations suivantes en cascade :
* Case 5 : `- 0x7e`
* Case 6 : `+ 0x7e`
* Case 7 : `- 0x7e`

Le total est `-0x7e`, ce qui correspond à `-126` en décimal. Le second argument doit correspondre à ce résultat pour éviter l'explosion.
Le mot de passe de la phase 3 est donc : `5 -126`

---

## Phase 4 : Récursion et parcours d'arbre dichotomique

### 1. Analyse Statique et Contraintes d'Entrée (`phase_4`)
L'inspection de la fonction `phase_4` dans WinDbg montre un appel à `sscanf` avec la chaîne de format `%d %d` (visible via le registre `rdx`). Le programme exige deux nombres entiers.

Avant de manipuler ces valeurs, le programme valide le domaine du premier argument :
* Il doit être compris entre `0` et `0xe` (14 en décimal).
* Si ces bornes ne sont pas respectées, `explode_bomb` est appelée.

Le programme prépare ensuite les arguments pour une fonction secondaire `func4` : `param_1` (entrée utilisateur), `0`, et `0xe`.
À la sortie, deux vérifications déterminent le succès : le retour de `func4` et le second entier saisi doivent tous deux être égaux à `10` (`0xa`).

![Le contenu de phase_4 dans Ghidra](Screenshots/Phase4_logic_Ghidra.png)

### 2. Analyse de la Fonction Récursive (`func4`)
La fonction `func4` implémente un parcours d'arbre binaire de recherche (dichotomie) :
1. **Calcul du milieu :** `mid = param_2 + (param_3 - param_2) / 2`.
2. **Branche gauche / droite :** Des appels récursifs s'additionnent au milieu selon si l'entrée est inférieure ou supérieure au pivot.
3. **Cas de base :** Si l'entrée égale le milieu, la fonction retourne directement la valeur.

![Le contenu de func4 dans Ghidra](Screenshots/Phase4_func4_decomp.png)

### 3. Résolution Mathématique
Pour que `func4(X, 0, 14)` retourne `10` :
* **Appel 1 :** Bornes `[0, 14]`, milieu = `7`. Reste à trouver : $10 - 7 = 3$.
* **Appel 2 (gauche) :** Bornes `[0, 6]`, milieu = `3`.
* Si $X = 3$, la fonction retourne `3` (cas de base). Le total remonte à $7 + 3 = 10$.

Le mot de passe pour désamorcer la phase 4 est : `3 10`

---

## Phase 5 : Arithmétique de pointeurs, tableaux statiques et graphes fonctionnels

### 1. Analyse Statique et Contraintes d'Entrée
La fonction `phase_5` attend deux entiers. Le premier subit un masquage binaire (`& 0xf`), forçant l'index à être compris entre 0 et 15.

![Décompilation de phase_5 dans Ghidra](Screenshots/Phase5_decomp.png)

### 2. Déréférencement Mémoire (`DAT_14001f1d0`)
Le cœur de la fonction utilise un déréférencement complexe pour parcourir un tableau. L'inspection de l'adresse de base du tableau en mémoire révèle les valeurs statiques suivantes :

| Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 |
| :---: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| **Valeur** | 10 | 2 | 14 | 7 | 8 | 12 | 15 | 11 | 0 | 4 | 1 | 13 | 3 | 9 | 6 | -- |

![Inspection mémoire du tableau statique](Screenshots/Phase5_memory_table.png)

### 3. Logique de l'Algorithme (Graphe orienté)
L'algorithme utilise chaque valeur du tableau comme index pour le saut suivant.
Pour réussir, il faut :
* Parcourir **exactement 15 itérations** avant de tomber sur la valeur `15`.
* Le second entier doit correspondre à la **somme cumulée** des 15 valeurs visitées.

### 4. Résolution
La seule chaîne permettant 15 sauts démarre à l'index `5` :
`5 -> 12 -> 3 -> 7 -> 11 -> 13 -> 9 -> 4 -> 8 -> 0 -> 10 -> 1 -> 2 -> 14 -> 6 -> [15]`
Somme : $12 + 3 + 7 + 11 + 13 + 9 + 4 + 8 + 0 + 10 + 1 + 2 + 14 + 6 + 15 = 115$

Le mot de passe pour désamorcer la phase 5 est : `5 115`

---

## Phase 6 : Listes chaînées, structures en mémoire et endianness

### 1. Analyse Statique et Contraintes d'Entrée
Le programme exige une suite de 6 entiers uniques compris entre 1 et 6 (une permutation).

![Décompilation de phase_6 dans Ghidra](Screenshots/Phase6_decomp.png)

### 2. Détection de Structure et Logique (Liste Chaînée)
En analysant les décalages mémoire (`+ 0x8`), l'algorithme manipule une liste chaînée :
```c
struct Node {
    int value;
    int id;
    struct Node *next;
};
```
Le programme reconstruit les liens de la liste selon l'ordre entré par l'utilisateur. La condition de victoire impose que la liste finale soit triée par **ordre strictement décroissant** de valeur.

### 3. Extraction des Données (Piège du Little-Endian)
L'inspection de la mémoire des 6 nœuds nécessite d'inverser les octets (architecture x86-64 Little-Endian) :

![Inspection mémoire des structures Node (Partie 1)](Screenshots/Phase6_nodes_memory_part1.png)
![Inspection mémoire des structures Node (Partie 2)](Screenshots/Phase6_nodes_memory_part2.png)

* `node1` : `12 02 00 00` -> `0x0212` -> 530
* `node2` : `c2 01 00 00` -> `0x01c2` -> 450
* `node3` : `15 02 00 00` -> `0x0215` -> 533
* `node4` : `93 03 00 00` -> `0x0393` -> 915
* `node5` : `a7 03 00 00` -> `0x03a7` -> 935
* `node6` : `00 02 00 00` -> `0x0200` -> 512

### 4. Résolution
Tri décroissant des valeurs : $935 > 915 > 533 > 530 > 512 > 450$
Le mot de passe pour désamorcer la phase 6 est : `5 4 3 1 6 2`

---

## Conclusion Globale du Lab

Le démontage de ce binaire a permis de mobiliser des concepts fondamentaux :
* **Phase 1 :** Extraction statique de chaînes et validation dynamique.
* **Phase 2 :** Analyse de tableaux en assembleur.
* **Phase 3 :** Identification d'un flux de contrôle `switch` (*fall-through*).
* **Phase 4 :** Reconstruction d'un algorithme récursif dichotomique.
* **Phase 5 :** Arithmétique de pointeurs et parcours de graphe orienté.
* **Phase 6 :** Identification de structures en mémoire (`struct`), listes chaînées et gestion de l'endianness x86-64.

### Bilan Personnel

![La bombe est totalement désamorcée](Screenshots/bomb_defused.png)

Voir ce message de réussite s'afficher dans la console a été un moment incroyablement gratifiant. Cet exercice m'a poussé hors de ma zone de confort. Il m'a appris à ne pas paniquer face à l'assembleur, à comprendre les comportements des compilateurs C, et à maîtriser des outils standards de l'industrie comme WinDbg et Ghidra.

Les moments de frustration sur les phases récursives ou l'arithmétique de pointeurs ont été compensés par l'immense satisfaction de comprendre la logique sous-jacente. Ce laboratoire m'a prouvé une chose essentielle : **j'aime profondément la Rétro-Ingénierie**. Je suis désormais prêt et motivé pour continuer l'aventure et m'attaquer à des défis encore plus complexes !