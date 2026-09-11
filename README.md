## Phase 1 : Reconnaissance statique et comparaison de chaînes

### 1. Reconnaissance initiale (Strings)
Avant même de désassembler le binaire, j'ai effectué une première passe de reconnaissance avec l'outil `strings` de Sysinternals pour extraire le texte en clair du fichier `bomb.exe`.
Parmi les résultats, plusieurs chaînes suspectes ressemblaient à des mots de passe potentiels, notamment :
* `passphrase`
* `DrEvil`
* `I am just a renegade hockey mom.`

### 2. Analyse Dynamique (WinDbg)
Pour confirmer quel mot de passe était le bon, j'ai choisi d'analyser la fonction `phase_1` dynamiquement sous WinDbg en lisant directement l'assembleur.
En exécutant le code pas à pas (`p`), j'ai repéré la préparation des arguments juste avant l'appel à la fonction de vérification (`strings_not_equal`). Le programme chargeait une adresse mémoire spécifique dans un registre.

Pour inspecter le contenu de cette adresse, j'ai utilisé la commande d'affichage de chaîne ASCII dans WinDbg :
`da [adresse_du_registre]`

La mémoire a révélé la chaîne exacte attendue par le programme, confirmant l'une de mes hypothèses issues de la reconnaissance initiale.

### 3. Solution
Le mot de passe pour désamorcer la phase 1 est : `I am just a renegade hockey mom.`

---

## Phase 2 : Boucles et structures de données (Tableaux)

### 1. Analyse Dynamique et Statique
L'analyse de la fonction `read_six_numbers` appelée au début de la phase 2 a révélé l'utilisation de `sscanf` avec un format nécessitant 6 entiers. En observant les décalages de 4 octets (`+4`, `+8`, etc...) sur la pile dans WinDbg, j'ai déduit que l'entrée était stockée dans un tableau de 6 entiers.

En passant sur Ghidra pour analyser la logique de validation, j'ai identifié deux conditions critiques :
* Le premier entier du tableau doit être égal à `1`.
* Une boucle itère sur les 5 entiers restants avec la logique suivante : `array[i] = array[i-1] * 2`.

### 2. Résolution
Chaque nombre doit être le double du précédent. En partant de 1, la suite logique est une progression géométrique de raison 2.
Le mot de passe de la phase 2 est : `1 2 4 8 16 32`

---

## Phase 3 : Flux de contrôle et Fall-through Switch

### 1. Reconnaissance des arguments (WinDbg)
J'ai d'abord exécuté la phase 3 dans WinDbg. En inspectant les arguments passés à `sscanf` avec la commande `da`, j'ai identifié la chaîne de format `%d %d`, indiquant que la bombe attendait deux nombres entiers.

### 2. Analyse de l'algorithme (Ghidra)
Le décompilateur a révélé une structure de type `switch` basée sur le premier entier fourni. L'analyse a mis en évidence deux contraintes majeures :
* Une vérification finale exige que le premier entier soit inférieur ou égal à 5. Les cas 6 et 7 sont donc des impasses.
* Le bloc `switch` ne contient aucun `break` avant le cas 7 (comportement de *fall-through*). Cela signifie que le programme exécute en cascade toutes les opérations mathématiques des cas inférieurs.

### 3. Résolution
J'ai choisi d'utiliser le cas `5`. Le premier argument est donc `5`.
La variable de validation démarre à 0 et subit les opérations suivantes en cascade :
* Case 5 : `- 0x7e`
* Case 6 : `+ 0x7e`
* Case 7 : `- 0x7e`

Le total est `-0x7e`, ce qui correspond à `-126` en décimal. Le second argument doit correspondre à ce résultat pour éviter l'explosion.
Le mot de passe de la phase 3 est donc : `5 -126`