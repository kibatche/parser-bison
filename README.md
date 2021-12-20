# Comprendre la grammaire shell avec Bison

### Introduction
Pour celles et ceux qui souhaiteraient faire minishell sans enchainer les ft_split comme des mojitos un soir de dépression, je vous propose ce petit tutoriel qui pourra aider des personnes qui s'intéressent a _comment_ sont gérées les étapes de tokénisation/parsing dans le shell.

Mais avant toute chose, ce tutoriel n'est pas une explication exhaustive de la considerable littérature théorique sur le sujet, ni une dissertation sur les avantages et les inconvénients de cette méthode. Il existe en effet de nombreuses discussions autour du parsing - tout à fait passionnantes - mais bien loin de mes connaissances. Cependant, concernant la partie théorique, je mettrai quelques liens vers des sites ou vidéos interessants sur tel ou tel aspect.

### Y autait il donc un besherell du Shell (un bereshell) ?

Excellente question ! En effet, si jamais vous ne connaissez pas du tout comment est géré le parsing, il peut sembler un peu étrange d'utiliser un mot tel que "grammaire".

Mais avant d'expliquer ce terme, intéressons-nous aux etapes qui conduisent a l'execution d'une commande.

- La tokénisation. C'est une étape cruciale, et probablement le première que vous mettrez en place. Elle consiste à découper en petites partie une ligne de commande selon des abstractions que vous aurez choisies ou reprises de celles du Shell. Pour reprendre un vocabulaire linguistique, "c'est la plus petite unité distinctive" permettant de dinstiguer une entité d'une autre. Cf [Phoneme](https://fr.wikipedia.org/wiki/Phon%C3%A8me "Phoneme") Par exemple pour :

`ls -la | wc -l`

La tokenisation resultante pourrait etre :

`WORD WORD PIPE WORD WORD`

Nous reviendrons sur cette assertion par la suite.

- Le parsing via la creation d'un AST. L'Abstract Syntax Tree est un arbre (ou non) qui consiste à prendre nos tokens et de déterminer si leur enchainement correspond à une syntaxe pré-déterminée par des règles. C'est cette syntaxe et sa compréhension que nous allons étudier dans ce tutoriel. Exemple d'AST :

![Exemple AST](https://github.com/Batche-Hub/parser-bison/blob/main/Screenshot_from_2021-12-06_12-10-17.png "Exemple AST")

Ces deux etapes, nous pouvons les réaliser grace a une grammaire.

Cette grammaire ressemble à cela (Source : [documentation Shell Command Language](https://pubs.opengroup.org/onlinepubs/009604499/utilities/xcu_chap02.html#tag_02_10_02 "documentation Shell COmmand Language")) :

```lua
<SNIP>

%token  WORD
%token  ASSIGNMENT_WORD
%token  NAME
%token  NEWLINE
%token  IO_NUMBER


/* The following are the operators mentioned above. */


%token  AND_IF    OR_IF    DSEMI
/*      '&&'      '||'     ';;'    */

<SNIP>

%start  complete_command
%%
complete_command : list separator
                 | list
                 ;
list             : list separator_op and_or
                 |                   and_or
                 ;
and_or           :                         pipeline
                 | and_or AND_IF linebreak pipeline
                 | and_or OR_IF  linebreak pipeline
<SNIP>
```

C'est particulierement ce type de fichier que nous allons étudier. Et pour répondre a la question, oui il existe un Bereshell, et il se trouve là ! [Shell](https://pubs.opengroup.org/onlinepubs/9699919799/ "Shell")

### Bison

Maintenant, le titre de ce document mentionne "avec Bison". En effet, pour des raisons pédagogiques, Bison est un outil extrêmement efficace afin de comprendre cette grammaire et ses implications.

Bison est un logiciel open source, disponible ici : [Bison](https://www.gnu.org/software/bison/ "Bison")

Je vous conseille fortement d'installer Bison à partir de ce site (et des fichiers compressés disponibles), car il contient une option qui nous sera très utile par la suite et qui semble absente des archives Ubuntu . Installez le, tout est expliqué au sein de l'archive dans le fichier "INSTALL".

### Comprendre la grammaire

Avant de nous lancer avec bison, revenons au fichier issu de la documentation du Shell.

Le document commence par ceci :

```lua
/* -------------------------------------------------------
   The grammar symbols
   ------------------------------------------------------- */


%token  WORD
%token  ASSIGNMENT_WORD
%token  NAME
%token  NEWLINE
%token  IO_NUMBER


/* The following are the operators mentioned above. */


%token  AND_IF    OR_IF    DSEMI
/*      '&&'      '||'     ';;'    */


%token  DLESS  DGREAT  LESSAND  GREATAND  LESSGREAT  DLESSDASH
/*      '<<'   '>>'    '<&'     '>&'      '<>'       '<<-'   */
<SNIP>
```

On retrouve ici nos entites insécables, nos tokens, la base du langage. On peut remarquer qu'il y en a beaucoup d'inutiles dans le cadre du projet minishell, on devra donc considérablement limiter ce vocabulaire.

Par exemple :

```lua
%token  WORD
%token  HEREDOC  DGREAT  LESS  GREAT
/*      '<<'      '>>'    '<'   '>'*/

%token  PIPE
/*      '|'  */
```

On ne garde que le token WORD et tous les operateurs dont nous avons besoin pour minishell. C'est bien plus court !

A la suite du fichier shell, nous avons une notation particulière . En effet, nous ne l'avons pas precisé, mais le fichier shell est en fait un fichier nomenclaturé comme un fichier Yacc.

Ainsi, les règles syntaxiques sont présentées sous la forme suivante :

```lua
cmd_name         : WORD 
							;
```
 Que concrètement cela veut dire ? Que l'entité abstraite cmd_name (à gauche), est composée du token WORD. Par ailleurs, on peut remarquer que d'autre entités abstraites peuvent être composées de plusieurs entités abstraite et/ou token, y compris elles-mêmes. Exemple :
 
```lua
io_redirect      :           io_file
                 | IO_NUMBER io_file
                 |           io_here
                 | IO_NUMBER io_here
                 ;
```

Cela veut dire que l'entité abstraite io_redirect peut etre composée par :

- io_file **OU**
- IO_NUMBER io_file **OU**
- io_here **OU**
- IO_NUMBER io_here

Et ces différentes possibilités de compositions sont separées par le caractere '|' (**OU**), l'ensemble se terminant systématiquement par ';'.

On voit donc que les entités abstraites sont des compositions. Et la raison en devient plus évidente à mesure qu'on comprend le fichier. Car en regarde bien, le début de la partie concernant la syntaxe, nous avons ceci :

```lua
%start  complete_command
%%
complete_command : list separator
                 | list
                 ;
```

On commence par notre complete_command, et si on la decompose avec les parties à droite (ses définitions) et que nous regardons par la suite en quoi sont définies ces parties et ainsi de suite, nous arriverions à un ou des tokens. Donc l'idée de cette grammaire, c'est de voir comment on peut arriver à l'entité "ultime" : une complete_command, c'est à dire une ligne de commande syntaxiquement correcte.

Si ce n'est pas très clair, imaginez des matriochkas qui dépilées suite à suite nous mèneraient à une dernière qui est une matriochka insécable. Si on fait le chemin inverse, c'est à dire les empiler les unes avec les autres selon les règles de notre grammaire, nous arriverions à une grosse matriochka qui serait le résultat final recherché.

Cependant, encore une fois, cette grammaire est très dense par rapport à ce dont nous avons besoin, et il faut donc tailler large. Ce travail est intéressant à faire, car cela nous permet d'apprivoiser la syntaxe. Plutôt que de ne pas mettre le fichier, je mets une version encore plus légère que celle dont vous aurez besoin pour minishell. Cela fera un excellent point de départ, en vous laissant toute latitude pour compléter et faire votre propre grammaire par la suita.

### La grammaire simplifiée ^ 2

Pour cet exemple, nous n'allons garder qu'un token : WORD. Et seulement quelques règles afférentes à ce token.

Le résultat est le suivant, à mettre dans un fichier ".y" afin de suivre le tutoriel  :

```lua
/* -------------------------------------------------------
Le token
   ------------------------------------------------------- */
%token  WORD

/* -------------------------------------------------------
La grammaire
   ------------------------------------------------------- */
%start program
%%
program          :                                            command
                 ;
command          :               cmd_name cmd_suffix
                 |                                                     cmd_name
                 ;
cmd_name         :                                                WORD
                 ;
cmd_suffix       :                                                  WORD
                 |                                       cmd_suffix WORD
                 ;
```

Reprenons le fichier.

Cette partie indique le token utilisé :

```lua
/* -------------------------------------------------------
Le token
   ------------------------------------------------------- */
%token  WORD
```

Cette partie concerne le départ de la grammaire (et par corolaire le résultat attendu si la syntaxe est correcte) :

```lua
/* -------------------------------------------------------
La grammaire
   ------------------------------------------------------- */
%start program
%%
```

Et enfin la dernière partie stipule les règles que nous avons mis en place, les explications sont en commentaires :

```lua
/*Un program est composé d une entité command */

program          :                                            command
                 ;
/*command peut être soit composée de cmd_name ET cmd_suffix, ou bien seulement de cmd_name*/
command          :               cmd_name cmd_suffix
                 |                                                     cmd_name
                 ;
/*cmd_name est composé du token WORD*/
cmd_name         :                                                WORD
                 ;
/*cmd_suffix est composé du token WORD ou bien d un autre cmd_suffix et du token WORD */
cmd_suffix       :                                                  WORD
                 |                                       cmd_suffix WORD
                 ;
```

### Comprendre les implications de cette grammaire à l'aide de bison

Seulement, c'est bien beau tout ca, mais on a du mal à voir à quoi ca pourra bien servir, car nous ce qui nous intéresse, c'est de coder. Mais avant tout, il nous reste à voir qu'est-ce que cette grammaire implique concrètement.

Pour ce faire, une fois le fichier "ma_grammaire.y" de fait, passez le a bison :

`bison --html ma_grammaire.y`

Cela nous sortira un fichier html, que vous pouvez ouvrir avec firefox.

Ce fichier récapitule toutes nos règles, nous permet de voir s'il y a des conflits dans nos règles, etc. En bref, c'est un condensé plutôt lisible d'informations concernant notre grammaire.

Si on se concentre sur la grammaire très simplifiée faite un peu plus haut, nous avons ceci en sortie html (je ne copie pas le début du fichier) :

```lua
Grammar
    0 $accept → program $end

    1 program → command

    2 command → cmd_name cmd_suffix
    3         | cmd_name

    4 cmd_name → WORD

    5 cmd_suffix → WORD
    6            | cmd_suffix WORD
<SNIP>
Automaton
State 0

    0 $accept → • program $end
    1 program → • command
    2 command → • cmd_name cmd_suffix
    3         | • cmd_name
    4 cmd_name → • WORD

    WORD  shift, and go to state 1

    program   go to state 2
    command   go to state 3
    cmd_name  go to state 4
State 1

    4 cmd_name → WORD •

    $default  reduce using rule 4 (cmd_name)
State 2

    0 $accept → program • $end

    $end  shift, and go to state 5
State 3

    1 program → command •

    $default  reduce using rule 1 (program)
State 4

    2 command → cmd_name • cmd_suffix
    3         | cmd_name •  [$end]
    5 cmd_suffix → • WORD
    6            | • cmd_suffix WORD

    WORD  shift, and go to state 6

    $default  reduce using rule 3 (command)

    cmd_suffix  go to state 7
State 5

    0 $accept → program $end •

    $default  accept
State 6

    5 cmd_suffix → WORD •

    $default  reduce using rule 5 (cmd_suffix)
State 7

    2 command → cmd_name cmd_suffix •  [$end]
    6 cmd_suffix → cmd_suffix • WORD

    WORD  shift, and go to state 8

    $default  reduce using rule 2 (command)
State 8

    6 cmd_suffix → cmd_suffix WORD •

    $default  reduce using rule 6 (cmd_suffix)

```

La première partie sont nos règles de grammaire plus joliement présentées que dans le fichier yacc.

La seconde partie est l'automaton, et c'est celle qui va nous aider à construire l'AST. En effet, elle détaille quel enchaînement doit se produire en fonction de la rencontre de tel ou tel token, de telle ou telle abstraction. Cela peut sembler un peu confus pour l'instant, mais vous verrez que la suite rendra tout cela plus clair !

### Shift et Reduce

Mais avant toute chose, intéressons-nous à deux mots : Shift et Reduce, que l'on retrouve tout le long de l'automaton.

Cela est directement en rapport avec le type de parser que nous utilisons : un parser "LR" ([Left Right Parser](https://fr.wikipedia.org/wiki/Analyseur_LR "Left Right Parser"))

Nous n'allons pas rentrer dans les détails, mais pour construire last, nous avons besoin de deux choses essentielles :

- Une stack sur laquelle empiler une entité.
- Une tête de lecture sur les tokens afin de savoir où nous en somme

Shift veut dire qu'on met telle entité ou token sur la stack (une pile), nous conduisant à un autre état (un "state").
Reduce veut qu'on réduit telle ou telle entite, tel token, ou groupe d'entité selon l'une des règles de notre grammaire. Une réduction vaut retour à l'état initial, l'état 0.

### Enchaînement

Ainsi pour Shift, a l'état 0 de mon fichier, si jamais j'ai un token WORD sur ma tête de lecture :

`WORD     shift, and go to state 1`

Veut dire qu'on met le token WORD sur la stack, qu'on avance la tete de lecture sur le buffer de token (on avance de 1 token, left -> right) et qu'on doit aller à l'état 1 pour connaitre l'action suivante.

Qui est celle-cii :

```lua
State 1

    4 cmd_name → WORD •

    $default  reduce using rule 4 (cmd_name)
```
On voit qu'il n'y qu'une seule regle : le token WORD devient l'entite cmd_name selon la règle n°4 qui stipule que :

` 4 cmd_name → WORD`

Si on représente la stack, elle est donc comme suit :

`STACK = cmd_name $`

Je reviens donc a mon état initial. Sauf que là, au lieu de considérer mon buffer, je vais maintenant considérer la stack, car elle n'est pas vide. C'est une chose important à savoir.

A l'etat 0, elle ne sera plus jamais vide dès lors qu'on a lu le premier token, car l'idée, si la synatxe est bonne, sera d'avoir l'entité ultime (un "program"), et si ce n'est pas le cas, cela veut dire que la syntaxe est fausse (ou bien votre grammaire mais espérons que ce ne soit pas le cas ^^).

Par exemple, mon état 0 c'est ça :

```lua
State 0

    0 $accept → • program $end
    1 program → • command
    2 command → • cmd_name cmd_suffix
    3         | • cmd_name
    4 cmd_name → • WORD

    WORD  shift, and go to state 1

    program   go to state 2
    command   go to state 3
    cmd_name  go to state 4
```

Si je reprends mon exemple si dessus, j'ai maintenant "cmd_name" dans ma stack, et on me dit qu'il faut que j'aille à l'etat 4.

```lua
State 4

    2 command → cmd_name • cmd_suffix
    3         | cmd_name •  [$end]
    5 cmd_suffix → • WORD
    6            | • cmd_suffix WORD

    WORD  shift, and go to state 6

    $default  reduce using rule 3 (command)

    cmd_suffix  go to state 7
```

Et ainsi de suite. Le mieux, est de prendre un exemple.

Avec ma grammaire simplifiée, pour la commande suivante :

`ls -l -a`
Donne les tokens suivants :

`WORD WORD WORD END`

Pourquoi le token "END" ? C'est un token particulier qui me permet de signifier la fin du buffer de token. C'est plus simple à gérer ainsi.

| Numéro d'étape| Etat courant  | Stack  | Buffer  | Etat suivant | Action |
| ------------| ------------ | ------------ | ------------ | ------------ | ------------ |
|1|  0 | vide  | WORD WORD  WORD END| 1 | SHIFT
|2| 1  | WORD  | WORD WORD END | 0 | REDUCE |
|3| 0  | cmd_name  | WORD WORD END | 6 | SHIFT
|4|  6 | cmd_name WORD  | WORD END  | 0 | REDUCE |
|5|  0 | cmd_name cmd_suffix | WORD END  | 4 | |
|6|  4 | cmd_name cmd_suffix   | WORD END  | 7 | |
|7| 7 | cmd_name cmd_suffix   | WORD END  | 8 | SHIFT |
|8| 8 | cmd_name cmd_suffix WORD  | END  | 0 | REDUCE |
|9| 0  | cmd_name cmd_suffix  | END  | 4 | |
|10| 4  | cmd_name cmd_suffix  | END  | 0 |  REDUCE |
|11| 0  | command | END  | 3 |  |
|12| 3  | command | END  | 0 |  REDUCE |
|13| 0  | program | END  | 2 |   |
|14| 2  | program | END  | 5 | SHIFT  |
|15| 5  | program END |   |  | Accept. Fin de l'analyse.  |

Ainsi, avec ce tableau récapitulatif (que vous pouvez reprendre grâce au fichier html, qui inclus des liens vers les états suivants et les règles), on voit que selon notre grammaire, la commande "ls -l -a", représentée par la suite de token "WORD WORD WORD END" est syntaxiquement correct ! Youhou !

Une chose intéressante à remarquer, c'est que les étapes 8 et 9 représentent bien le fait qu'une entité peut devenir elle-même si elle est composée d'une autre entité en plus d'elle même. Ainsi cmd_suffix devient de façon abstraite lui-même s'il est composé d'un token WORD en plus. Les deux entités sont réduites dans un simple cmd_suffix.

### Mais que faire de cela ?

C'est une question complexe à répondre. Car l'idée, c'est de faire un AST, et là même si dans l'idée nous avons tout ce qu'il faut pour faire cela, nous n'avons pas encore codé.

Le problème, quelque part, c'est qu'il y a plein de manière de mettre en place ces règles et leurs conséquences. Au lieu de faire un "vrai" AST, on peut faire une liste chaînée de commande. Car au final, cette abstraction là ne nous induit pas forcément à la construction d'un arbre binaire. En somme, on peut très bien coder cet automaton, et générer un produit "final" (une suite de commande à exécuter) de façon indépendante de l'analyse syntaxique.

Plus concrètement, pour ma part j'ai fait la chose suivante :

- J'ai une state machine, qui reproduit toutes les étapes de l'automaton en code. Donc j'ai codé tous les états, puis les résultats possible etc. C'est une sorte de récursion géante qui évalue à chaque moment la validité de l'input selon les règles.

- Sauf qu'en parallèle, je crée une liste chaînée de commandes, au fur et à mesure de la validation de la syntaxe. Ainsi, dès que j'aie une commande complète (par exemple "ls -l -a"), je crée un élément dans la liste, qui si l'ensemble est validé, sera retourné et utilisable en vue de l'exécution.

- On peut aussi créer un arbre. Là, ce n'est pas forcément plus compliqué, mais l'idée est de trimballer les noeuds de l'arbre à chaque étape et de les connecter correctement (c'est une gymnastique). Faire directement le travail grâce à l'analyse facilite grandement les choses, car naturellement le travail d'analyse conduit à un arbre qu'on pourra par la suite exécuter facilement (voir l'exemple au tout début du tutoriel).

- Je ne l'ai pas fait car il n'y a pas d'intérêt de faire un arbre pour notre minishell (hormis l'intérêt de voir les arbres, et peut-être si on fait les bonus avec les évaluations '||' et '&&' mais c'est la galère à cause de la gestion des parenthèses), et que pour des raisons de temps il me semblait plus simple de mettre en place une liste.

J'espère néanmoins que ce document vous permettra de vous faciliter le début du chemin.
