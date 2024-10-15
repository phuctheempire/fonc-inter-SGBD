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
- Fichier: Contient des blocs

## Structure indexation

1. Index:
- Concept: In a database, an index is a **data structure** that *improves the speed of data retrieval* operations on a table at the cost of additional writes and storage space.
- Informatiques: C'est un fichier qui permet de trouver un enregistrement dans une table.
    - **Clé d'indexation** = **une liste** *d'un ou plusieurs attributs*.
    - **Une adresse** (déjà vu) est *une adresse de bloc ou une adresse d'enregistrement*.
    - Entrée d'index : enregistrements **de la forme [valeur, addr]**, valeur est une valeur de clé.
    - L'index est **trié** sur valeur
- *L'index ne sert à rien pour toute recherche ne portant pas sur la clé.*
- Types d'index:
    - Index non dense: le fichier de données est **trié sur la clé**, comme un dictionnaire. *L'index ne référence que la première valeur de chaque bloc*.
    ![alt text](../assets/images/nondense.png)
        - Operations:
            - Par clé
            - Par intervalle
            - Par recherche de préfixe

        ```
            Exemple concret
            Sur notre chier de 1,2 Go
             En supposant qu'un titre occupe 20 octets, une adresse 8 octets
             On a toujours 300.000 blocs pour stocker l'intégralité de la base
             Taille de l'index : 300 000 ∗ (20 + 8) = 8, 4Mo octets
            Processus de lecture (index sur le titre du lm) :
            1- On parcourt l'index (qui peut être en mémoire !) pour trouver l'adresse du bloc
            2- On lit le bloc en question
            3- On parcourt en mémoire le bloc pour trouver le lm
            Le coût en terme d'I/O disque est O(1).
            Beaucoup plus petit que le chier !
            Problème : maintenir l'ordre sur le chier et sur l'index.
        ```

    - Index dense: *L'index référence chaque enregistrement*.
        ![alt text](../assets/images/dense.png)
        - Fichier de données **non trié**. **Toutes les valeurs de clé sont représentées**
        - Engendre des accès aléatoires
        ```
            Exemple concret
            Sur notre chier de 1,2 Go
             Une année = 4 octets, une adresse 8 octets
             Taille de l'index : borné par 1 000 000 ∗ (4 + 8) = 12 Mo (si chaque lm a une
            année diérente), ou plus précisément 1 000 000 ∗ 8 + |annes| ∗ 4
            Encore 100 fois plus petit que le chier.
            Recherches :
             Par clé : comme sur un index non-dense
             Par intervalle (exemple [1950, 1979]) :
            recherche, dans l'index de la borne inférieure
            parcours séquentiel dans l'index
            à chaque valeur : accès au chier de données
            Engendre des accès aléatoires.
        ```
    - Index multi-niveaux
        ![alt text](../assets/images/multiniveau.png)
        - Essentiel : l'index est trié, donc on peut l'indexer par un second niveau *non-dense*
        - Si tous les niveaux étaient denses, ils auraient tous la même taille *O(n)* où *n* est le nombre d'enregistrements -> il faut que l'index soit trié.
- L'index **accélère les requêtes de recherche**, mais **ralentit les requêtes de mise à jour**. ( *il a un cout* )

## Arbres B+

- Un noeud feuille est un index dense local, contenant des entrées d'index.
- Chaque entrée référence un (ou plusieurs) enregistrements du chier de données : celui (ceux) ayant la même valeur de clé
que l'entrée
![alt text](../assets/images/ex-arbre.png)
