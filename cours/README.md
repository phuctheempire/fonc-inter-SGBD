# Fichiers et Indexation

## Gestion des mémoires

- La structure hierarchique des fichiers est gérée par le système de fichiers:
![Hierarchie des fichiers](../assets/images/memoire.png)

- Operation de lecture: 
    - Si la donnée est fait partie du cache, elle est retournée directement ( *le SGBD prend le bloc, accède à la donnée et la retourne* )
    - Sinon, le SGBD accède au disque pour lire le bloc, le met en cache et retourne la donnée
    ![Lecture](../assets/images/operation_de_lecture.png)

- Hypothèse de localité: le SGBD garde en mémoire les blocs après utilisation, afin d'exploiter à la fois
    - la **localité spatiale** : les autres données du bloc.
    - la **localité temporelle** : la donnée sera probablement réutilisée.

- Le **hit ratio**: Le paramètre qui mesure l'efficacité d'une mémoire cache
    - $$ \text{Hit Ratio} = \frac{\text{nb\_lec\_logique} - \text{nb\_lec\_physique}}{\text{nb\_lec\_logique}} $$
    - If nb_lec_logique = nb_lec_physique ( *toutes les lectures logiques (demande de bloc) aboutissent à une lecture physique (accès au disque)* ), alors le hit ratio = 0
    - If nb_lec_physique = 0 ( *toutes les lectures logiques sont satisfaites par le cache* ), alors le hit ratio = 1
    - En pratique. Un bon Hit Ratio est supérieur à 80%, voire 90% : presque toutes les lectures se font en mémoire !
    - *Attention*: nb_lec_logique > nb_lec_physique, .
    - Pour avoir un bonne hit ratio:
        - Il faut allouer le plus de mémoire possible au SGBD
        - Limiter la taille de la base
        - Il faut que les données utiles soient en mémoire centrale: **certaines parties de la base sont beaucoup plus lues que d'autres**
- Stratégie de remplacement: LRU (Least Recently Used)
    ![LRU](../assets/images/LRU.png)
    - Conséquence : le contenu du cache est une image fidèle de l'activité récente sur la base de données.
- Operation mise à jour:
    - Approche naive: 
    - **Logging**: Optimal idea
        - For every update, record infos to allow to redo the update in case of failure
            - Sequential write to log file ( Seperate disk )
        - Log: An **ordered list** of all updates
        - Transactions are not considered complete until the corresponding changes are safely recorded in the write-ahead log.
        - Write ahead logging:
            - Force the log record for an update before the update itself
            - Sequential write to log file
        - Write to memory after log record
        - Write to disk ( opportunistically and randomly )
    ![alt text](../assets/images/log.png)

- Principe de localité: l'ensemble des données utilisées par une application pendant une période donnée forme souvent un groupe bien identifié.
    - **Localité spatiale** : si une donnée d est utilisée, les données proches de d ont de fortes chances de l'être également.
    - **Localité temporelle** : quand une application accède à une donnée d, il y a de fortes chances qu'elle y accède à nouveau peu de temps après.
    - **Localité de référence** : si une donnée d1 référence une donnée d2, l'accès à d1 entraîne souvent l'accès à d2.
### Questions:
- Que contient le cache ?
    - une copie de certains blocs de la base
    - des données d'une application utilisant le SGBD
- Un cache peut-il être plus grand que la base sur le disque ?
    - oui, et ça peut être utile
- Peut-il y avoir plus de lectures physiques que de lectures logiques ?
    - oui
- Une mise à jour se fait :
    - dans le cache, sans transfert immédiat
- Quel est le danger d'effectuer une mise à jour dans le cache et pas sur le disque ?
    - la mise à jour peut être perdue en cas de panne
- Parmi les arguments contre l'écriture immédiate d'un bloc contenant un enregistrement modifié, lequel vous semble faux ?
    - on évite d'écrire un bloc complet à chaque modification
    - on peut modifier un même enregistrement plusieurs fois de suite sans avoir à écrire sur le disque
    - on évite d'écraser des mises à jour faites par d'autres applications
    - **on peut changer d'avis et revenir à la version stockée sur le disque**



## Organisation des fichiers

- Enregistrements:
    - Un enregistrement = une suite de *champs* stockant les valeurs des attributs.
    ![](../assets/images/tpyes.png)
    - Certains champs ont une taille variable ; d'autres peuvent être à NULL : pas de valeur.
    - En-tete:
        - Exemple: 
            table Film (id INT, titre VARCHAR(50), année INT)
            Enregistrement (123, 'Vertigo', NULL)
            ![](../assets/images/entete.png)
            Contient: La **longueur de l'enregistrement** (entier sur *4 octets*, chaine de caractère de *7 octets*, longueur de la chaine sur *1 octet* **7+4+1=12**) puis **un masque indiquant** que le 3e attribut est NULL. Le contenu de l'enregistrement se décrypte au moment de la lecture.
- Bloc: Contients des enregistrement, l'addresse, ... ( a une structure indexation )
    - Exemple:
        ![alt text](image.png)
        - Avec le schéma précédent, l'adresse d'un enregistrement est constituée
            - D'une adresse physique, celle du bloc. Ex : F1.12
            - D'une adresse logique, interne au bloc. Ex : 16
    - What happen if an enregistrement is too large to fit in a block ?
        - **Un enregistrement s'agrandit, mais qu'il reste de la place dans le bloc**: On modifie l'organisation interne ; la table locale d'adressage est modifiée
        - Un enregistrement s'agrandit, plus de place dans le bloc: on déplace l'enregistrement et on crée un chaînage dans l'en-tête du bloc
        - La création de chaînage **pénalise** les performances
        ![alt text](image-1.png)
    
- Fichier: Contient des blocs
    - Si les blocs sont très dispersés sur le disque, on aboutit à une fragmentation très pénalisante.

### Questions:
- Quelle est la différence entre un champ de type varchar(25) et un champ de type varchar(250) ?
    - le système refusera de stocker dans le premier une chaîne de plus de 25 octets
    - l'entête de l'enregistrement sera plus volumineuse pour le second
    - **le premier occupe 10 fois moins de place**
- Je représente l'adresse d'un enregistrement par son numéro de bloc B, et son numéro interne au bloc i (schéma d'indirection, vu ci-dessus). Quelle réponse est vraie ?
    - je peux aller lire directement l'enregistrement sur le disque
    - l'enregistrement est toujours en position i dans le bloc
    - **je dois d'abord lire le bloc avant de trouver l'enregistrement**

## Structure indexation

1. Index:
- Concept: In a database, an index is a **data structure** that *improves the speed of data retrieval* operations on a table at the cost of additional writes and storage space.
- Informatiques: C'est **un fichier** qui permet de trouver un enregistrement dans une table.
    - **Clé d'indexation** = **une liste** *d'un ou plusieurs attributs*.
    - **Une adresse** (déjà vu) est *une adresse de bloc ou une adresse d'enregistrement*.
    - Entrée d'index : enregistrements **de la forme [valeur, addr]**, valeur est une valeur de clé.
    - L'index est **trié** sur valeur
- Exemples:
    - Clés de recherche :
        - Le titre du film (c'est aussi la clé primaire)
        - l'année du film
    - Opérations :
        - Rechercher Vertigo
        - Rechercher les films parus entre 1960 et 1975
        - Rechercher les films commençant par 'V'
- *L'index ne sert à rien pour toute recherche ne portant pas sur la clé.*
- Types d'index:
    - Index non dense: le fichier de données est **trié sur la clé**, comme un dictionnaire. *L'index ne référence que la première valeur de chaque bloc*.
        ![alt text](../assets/images/nondense.png)
        - Hypothèse : le **fichier de données** est **trié** sur la clé, comme un dictionnaire. L'index ne référence que **la première valeur** de chaque bloc.
        - Operations:
            - Par clé: *je recherche Shining*
            - Par intervalle: *tous les films entre Greystocke et Psychose.*
            - Par recherche de préfixe: Par préfixe : tous les films commençant par 'M'.
            - Exemple concret
                ![alt text](image-2.png)
                Sur notre fichier de 1,2 Go
                - En supposant qu'un titre occupe 20 octets, une adresse 8 octets
                - Un bloc de 4 Ko
                - On a toujours 300.000 blocs pour stocker l'intégralité de la base
                - **Taille de l'index** : 300 000 ∗ (20 + 8) = 8, **4Mo** octets
                Processus de lecture (index sur le titre du film) :
                1- On parcourt l'index (qui peut être en mémoire !) pour trouver l'adresse du bloc
                2- On lit le bloc en question
                3- On parcourt en mémoire le bloc pour trouver le film
                Le coût en terme d'I/O disque est O(1).


    - Index dense: *L'index référence chaque enregistrement*.
        ![alt text](../assets/images/dense.png)
        - Fichier de données **non trié**. **Toutes les valeurs de clé sont représentées**
        - Engendre des accès aléatoires
        - Exemple concret
            ![alt text](image-5.png)
            Sur notre fichier de 1,2 Go
            - Une année = 4 octets, une adresse 8 octets
            - Taille de l'index : borné par 1 000 000 ∗ (4 + 8) = 12 Mo (si chaque film a une année différente), ou plus précisément 1 000 000 ∗ 8 + |annes| ∗ 4
            Encore 100 fois plus petit que le fichier.
            Recherches :
            - Par clé : comme sur un index non-dense
            - Par intervalle (exemple [1950, 1979]) :recherche, dans l'index de la borne inférieure parcours séquentiel dans l'index
            à chaque valeur : accès au fichier de données
            Engendre des accès aléatoires.
        ```
    - Index multi-niveaux
        ![alt text](../assets/images/multiniveau.png)
        - Essentiel : l'index est trié, donc on peut l'indexer par un second niveau *non-dense*
        - Si tous les niveaux étaient denses, ils auraient tous la même taille *O(n)* où *n* est le nombre d'enregistrements -> il faut que l'index soit trié.
        - Arrêt quand racine constitué d'un seul bloc.
        - Structure hiérarchique ; recherche de haut en bas.
- L'index **accélère les requêtes de recherche**, mais **ralentit les requêtes de mise à jour**. ( *il a un cout* )

### Questions:
- Qu'appelle-t-on une entrée d'index ?
    - un enregistrement de l'index, de la forme (clé, adresse)
    - l'intervalle de valeur couvert par les clés d'un même bloc
    - l'intervalle entre deux clés de l'index
    - le premier bloc de l'index
- Combien d'index non denses peut-on créer sur un fichier de données
    - au plus 1
    - exactement 1
    - autant qu'on veut
- Quel type d'index est mieux adapté à un dictionnaire ?
    - dense
    - non-dense
- Je dispose d'un index non-dense sur le titre des films. Est-ce que cet index me sera utile pour exécuter la requête SELECT * FROM FILMS WHERE TITRE <> "Matrix"
    - oui
    - non

## Arbres B+

### Concept
Aboutissement des structures d'index basées sur l'ordre des données
- c'est un arbre équilibré
- chaque n÷ud est un index local
- il se réorganise dynamiquement
![alt text](image-6.png)
- Un noeud feuille est un index dense local, contenant des entrées d'index.
- Chaque entrée référence un (ou plusieurs) enregistrements du chier de données : celui (ceux) ayant la même valeur de clé
que l'entrée
![alt text](../assets/images/ex-arbre.png)
- Un n÷ud interne est un index non dense local, les enregistrements servant de clé, intercalés avec des pointeurs.
![alt text](../assets/images/comp-noeud.png)
- Exemple:
    - Fichier 1M de films
    - Admettons qu'une **entrée d'index** occupe 12 octets, soit 8 octets pour l'adresse, et 4 pour la clé (l'année du film).
    - Chaque bloc contient 4 096 octets
    - On place donc 4096/12 = 341 **entrées** (au maximum) *dans un bloc*.
    - Il faut 1000000/341 = 2932 **blocs** pour le niveau des feuilles
    - Le deuxième niveau est non dense. Il comprend autant d'entrées que de blocs à indexer, soit 2932. Il faut donc 2932/341 = 8 blocs (au mieux).
    - Finalement, un troisième niveau, constitué d'**un bloc avec 8 entrées** sufft pour compléter l'index.
![alt text](image-7.png)
### Insert:
Supposons que le bloc contient seulement 2 entrées.
![alt text](image-8.png)
![alt text](image-9.png)


### Recherche
- Recherche par clé
    ```sql
    SELECT * FROM Film WHERE titre = 'Manhattan';
    ```
    - on lit la racine de l'arbre : Manhattan étant situé dans l'ordre lexicographique entre Easy Rider et Psychose, on doit suivre le chaînage situé entre ces deux titres ;
    - on lit le bloc intermédiaire, on descend à gauche ;
    - dans la feuille, on trouve l'entrée correspondant à Manhattan ;
    - il reste à lire l'enregistrement.

- recherche par intervalle
    ```sql
    select *
    from Film
    where annee between 1960 AND 1975
    ```
    - On fait une recherche par clé pour l'année 60
    - on parcourt les feuilles de l'arbre en suivant le chaînage, jusqu'à l'année 1975
    - à chaque fois on lit l'enregistrement
- Recherche par préfixe
    ```sql
    SELECT *
    FROM Film
    WHERE titre LIKE 'M\%'
    ```
    
    Revient à une recherche par intervalle.
    ```sql
    SELECT *
    FROM Film
    WHERE titre BETWEEN 'MAAAAAA...' AND 'MZZZZZZ...'
    ```
    Contre-exemple :
    ```sql
    SELECT *
    FROM Film
    WHERE titre LIKE '\%e'
    ```
    Ici index inutilisable


### Creation arbre B+
- Sur la clé primaire.

<!-- ```sql
create table Film (titre varchar(30) not null,
...,
primary key (titre)
);
``` -->

- Sur n'importe quel attribut.

```sql
create index filmAnnee on Film (annee)
```

Le SGBD synchronise le contenu de la table et celui de l'index
    
### Efficacité de l'arbre B+
Il est (presque) parfait !
- On a très rarement besoin de plus de trois niveaux
- Le coût d'une recherche par clé est le nombre de niveaux, plus 1.
- Supporte les recherches par clé, par intervalle, par préfixe
- Dynamique
On peut juste lui reprocher d'occuper de la place. Il existe aussi les arbres B qui
sont comme les arbres B+ sauf qu'on peut mettre un pointeur dans des
noeuds internes et pas seulement dans les feuilles (cf TD).

### Questions:

- Combien peut-on créer d'index en forme d'arbre B+ sur une table ?
    - un seul, car l'arbre B+ est non dense
    - deux, celui sur la clé primaire, et celui sur un autre attribut au choix
    - un pour chaque clé (primaire et étrangère)
    - autant que l'on veut
- Le chaînage des feuilles est utile pour ?
    - la procédure d'éclatement des feuilles
    - le maintien de la cohérence globale de l'arbre
    - la procédure de recherche par intervalle

- La recherche pour une valeur de clé, dans un arbre B+
    - s'effectue en suivant un unique chemin de la racine vers une feuille
    - s'effectue en parcourant séquentiellement les feuilles
    - s'effectue en trouvant le premier bloc, interne ou feuille, dont une entrée correspond à la valeur de la clé recherchée


## Le hachage
- Concurrent avec l'arbre B+
    - Meilleur (un peu, et en théorie) pour les recherches par clé
    - N'occupe aucune place
    - Se réorganise difficilement
    - Ne supporte pas les recherches par intervalle

- Principle: Le stockage est organisé en **N fragments (buckets)** constitués de séquences de blocs.
    - La répartition des enregistrements se fait par un **calcul**
    - Une **fonction de hachage h**: **prend une valeur de clé en entrée** et **retourne une adresse de fragment en sortie**
        - Stockage : on calcule l'adresse du fragment d'après la clé, on y stocke l'enregistrement
        - Recherche (par clé) : on calcule l'adresse du fragment d'après la clé, on y cherche l'enregistrement

- Exemple: 
    - On veut créer une structure de hachage pour nos 16 films
    - Chaque fragment fait un bloc et contient 4 enregistrements au plus
    - on alloue 5 fragments (pour garder une marge de man÷uvre)
    - un répertoire à 5 entrées (0 à 4) pointe vers les fragments
    - On définit la fonction h(titre) = rang(titre[0]) mod 5
        - Donc on prend la première lettre du titre (par exemple 'I' pour 'Impitoyable'), on prend sot rang dans l'alphabet (ici 9) et on garde le reste de la division par 5, le nombre de fragments.
        *h('Impitoyable') = 9 mod 5 = 4*
    ![alt text](image-10.png)
- Recherche compatible:
    - Par clé , **Oui**
        *SELECT * FROM Film WHERE titre = 'Impitoyable'*
    - Par préfixe ? **Non**
        *SELECT * FROM Film WHERE titre LIKE 'Mat%'*
    - Par intervalle : **non !** 
        *SELECT *
        FROM Film
        WHERE titre BETWEEN 'Annie Hall' AND 'Easy Rider'*
- Très important : h doit répartir **uniformément** les enregistrements dans les n fragments
- Difficulté: si une majorité de films commence par **une même lettre** (*'L' par exemple*) la répartition va être déséquilibrée.
- Plus grave : La structure simple décrite précédemment n'est pas dynamique.
    - On ne peut pas changer un enregistrement de place
    - Donc il faut créer un chaînage de blocs quand un fragment déborde
    - Et donc les performances se dégradent...
    ![alt text](image-11.png)
### Question

À quoi sert le répertoire de la structure de hachage ?
R0 À stocker les enregistrements d'après leur clé c
R1 À stocker l'adresse des enregistrements d'après la valeur de h(c)
R2 À stocker l'adresse des fragments d'après la valeur de h(c)

Ma clé primaire est un identifiant séquentiel. Que peut-on dire du stockage de deux
enregistrements consécutifs ?
R0 Les emplacements des enregistrements sont complètement indépendants.
R1 Ils sont forcément dans deux fragments distincts.
R2 Ils sont dans le même fragment, ou au pire dans deux fragments ajacents.

Quelle est, dans le pire des cas, le coût d'une recherche par clé dans une structure de
hachage statique ?
R0 Dans tous les cas une lecture suffit.
R1 Dans le pire des cas il faut deux lectures, une pour le fragment principal, l'autre
pour le fragment chaîné.
R2 Dans le pire des cas, on aura |base|/|fragment| lectures à effectuer.
R3 Dans le pire des cas tous les fragments sont chaînés et le coût est identique au
parcours du fichier.

## Le hashage extensible
- Le hachage extensible permet de réorganiser la table de hachage en fonction des insertions et suppressions.
- Principe la fonction de hachage h est fixe, mais on utilise les **n premiers bits** du
résultat h(c) pour s'adapter à la taille de la collection.
- Par rapport à la version de base du hachage, on ajoute deux contraintes:
    - Nombre d'entrées dans le répertoire est une puissance de 2
    - Fonction h donne toujours un entier sur 4 octets (32 bits)
- Exemple:
    ![alt text](image-12.png)
    - Supposons 3 films par fragment. L'insertion de Psychose (valeur 01110011) entraîne le débordement du premier fragment.
        - On double la taille du répertoire : quatre entrées 00, 01, 10 et 11.
        - On alloue un nouveau fragment pour l'entrée 01
        - Les entrées 10 et 11 pointent sur le même fragment
    - Le répertoire grandit, mais dans l'espace de stockage on ajoute seulement un nouveau fragment.
    ![alt text](image-13.png)
    Cas 1 : on insère dans un fragment plein, mais plusieurs entrées pointent dessus ⇒ on alloue un nouveau fragment, et on répartit les pointeurs
    ![alt text](image-14.png)
    Cas 2:  on insère dans un fragment plein, associé à une seule entrée ⇒ on double à nouveau le nombre d'entrées
    ![alt text](image-15.png)
- Le hachage extensible résout en partir le principal défaut du hachage, l'absence de dynamicité.
- Le répertoire tend à croître de manière exponentielle, ce qui peut soulever un problème à terme.
- Il reste une structure plaçante qui doit être complétée par l'arbre B pour des index secondaires.

### Question
Qu'est-ce qui caractérise le hachage extensible ?
R0 Les fragments sont de taille variable
R1 Le répertoire est de taille variable
R2 La fonction de hachage change régulièrement

Combien ajoute-t-on de fragments au moment quand l'un d'eux déborde ?
R0 1
R1 2
R2 p où p est le nombre d'entrée courant dans le répertoire
R3 2^(p−1) où p est le nombre d'entrée courant dans le répertoire
R3 2^p où p est le nombre d'entrée courant dans le répertoire

Combien d'entrées du répertoire peuvent référencer le même fragment ?
R0 1
R1 2
R3 log(p) où p est le nombre d'entrée courant dans le répertoire
R4 p/2 où p est le nombre d'entrée courant dans le répertoire
R5 p où p est le nombre d'entrée courant dans le répertoire


# Execution - Optimisation requetes

- Une requête SQL est **déclarative**. Elle ne dit pas **comment** calculer le résultat
- Dans un SGBD le *programme* qui exécute une requête est appelé **plan d'exécution**.
- **Plan execution**: c'est un **arbre**, constitué *d'opérateurs*.
- Deux étapes :
    - (A) plan d'exécution **logique** (l'algèbre) ;
    - (B) plan d'exécution **physique** (opérateurs).
    ![alt text](../assets/images/planexec.png)
- Optimisation: À chaque étape, plusieurs choix. Le système les évalue et choisit le meilleur.
![](../assets/images/opti.png)


## Operateur physiques:
### **Operateur**: 
- Forme **generique**:
- Avoir un tache
- **Bloquant** et **non bloquant**
### Blocage: 
- **Materielisation**: stocker le résultat d'une requête intermédiaire
![](../assets/images/materialisation.png)
- **Pipelinage**: 
![](../assets/images/streaming.png)
- **Temps réponse, temps d'execution**:
![](../assets/images/temp.png)
- Exemple:
    - **Blocage**: 
        - Tri (Order by)
        - Partitionnement (Group by) avec fonction d'agrégation (max, min, sum, avg)
        - Elimination de doublons (distinct) (opérateur dit semi-bloquant)
    - **Non blocage**:

### FullScan
- Au moment du open(), le curseur est positionné avant le premier nuplet
![alt text](image-16.png)
- Le premier next() entraîne l'accès au premier bloc, placé en mémoire.
![alt text](image-17.png)
- Le curseur se place sur le premier nuplet, qui est retourné comme résultat. Le temps de réponse est minimal.
- Le deuxième next() avance d'un cran dans le parcours du bloc.
![alt text](image-18.png)
![alt text](image-19.png)
![alt text](image-20.png)
### Execution une requete = avancer un curseur
![alt text](image-21.png)
### Notion iterateur
Chaque opérateur est implanté sous forme d'un itérateur.
Trois fonctions :
- **open** : initialise les tâches de l'opérateur ; positionne le curseur au début du résultat à fournir ;
- **next** : ramène l'enregistrement courant se place sur l'enregistrement suivant ;
- **close** : libère les ressources ;
Connexion :
- Un itérateur consomme des tuples d'un ou deux autres itérateurs.
- Un itérateur produit des tuples pour un autre itérateur (ou pour l'application).
Exemple:
![alt text](image-22.png)
![alt text](image-23.png)
![alt text](image-24.png)
## Some Operateur:

### IndexScan
- Implementation
![](../assets/images/imp_par_index.png)
- Execution:
![](../assets/images/index-par.png)
- Code:
![alt text](image-25.png)

### DirectAccess
- Implementation:
    - open(): Rien à faire
    - next(): Recoit addresse, access le bloc, utilise address local pour trouver le tuple et retourne un n-uplet

### Tri par fusion
![Tri par fusion](../assets/images/tri-fus.png)
- Operation:
    - Non bloquant
    - open(): Produire resultats ( Collection des tuples trié (*Sur disque ou memoire ?* ) )
    - next(): ne fait que lire, un à un, les tuples dans le résultat du tri


### Jointure
