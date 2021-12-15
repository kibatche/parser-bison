# Comprendre la grammaire shell avec Bison

### Introduction
Pour celles et ceux qui souhaiteraient faire minishell sans enchainer les ft_split comme des mojitos un soir de depression, je vous propose ce petit tutoriel qui pourra aider des personnes qui s'interessent a _comment_ sont gerees les etapes de tokenisation/parsing dans le shell.

Mais avant toute chose, ce tutoriel n'est pas une explication exhaustive de la considerable litterature theorique sur le sujet, ni une dissertation sur les avantages et les inconvenients de cette methode. Il existe en effet de nombreuses discussions autour du parsing - tout a fait passionnantes - mais bien loin de mes connaissances. Cependant, concernant la partie theorique, je mettrai quelques liens vers des sites ou videos interessants sur tel ou tel aspect.

### Y autait il donc un besherell du Shell (un bereshell) ?

Excellente question ! En effet, si jamais vous ne connaissez pas du tout comment est gere le parsing, il peut sembler un peu etrange d'utiliser un mot tel que "grammaire".

Mais avant d'expliquer ce terme, interessons-nous aux etapes qui conduisent a l'execution d'une commande.

- La tokenisation. C'est une etapes cruciale, et probablement le premiere que vous mettrez en place. Elle consiste a decouper en petites partie une ligne de commande selon des abstractions que vous aurez choisies ou reprises de celles du Shell. Pour reprendre un vocabulaire linguistique, "c'est la plus petite unite disinctive" permettant de dinstiguer une entite d'une autre. Cf [Phoneme](https://fr.wikipedia.org/wiki/Phon%C3%A8me "Phoneme") Par exemple pour :

`ls -la | wc -l`

La tokenisation resultante pourrait etre :

`WORD WORD PIPE WORD WORD`

Nous reviendrons sur cette assertion par la suite.

- Le parsing via la creation d'un AST. L'Abstract Syntax Tree est un arbre (ou non) qui consiste a prendre nos differents nos differentes unite insecable et de determiner si leur enchainement correspond a une syntaxe pre determine par des regles. C'est cette syntaxe et sa comprehension que nous allons etudier dans ce tutoriel. Exemple d'AST :

![Exemple AST](https://github.com/Batche-Hub/parser-bison/blob/main/Screenshot_from_2021-12-06_12-10-17.png "Exemple AST")

C'est deux etapes, nous pouvons les realiser grace a une grammaire.

Cette grammaire ressemble a cela (Source : [documentation Shell Command Language](https://pubs.opengroup.org/onlinepubs/009604499/utilities/xcu_chap02.html#tag_02_10_02 "documentation Shell COmmand Language")) :

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

C'est particulierement ce type de fichier que nous allons etudier. Et pour repondre a la question, oui il existe un Besherell, et il se trouve la ! [Shell](https://pubs.opengroup.org/onlinepubs/9699919799/ "Shell")

### Bison

Maintenant, le titre de ce document mentionne "avec Bison". En effet, pour des raisons pedagogique, Bison est un outil extremement efficace afin de comprendre cette grammaire et ses implications.

Bison est un logiciel open source, disponible ici : [Bison](https://www.gnu.org/software/bison/ "Bison")

Je vous conseille fortement d'installer Bison a partir de ce site (et des fichiers compresses disponibles), car il contient une option qui nous sera tres utile par la suite et qui semble absente des archive Ubuntu (au moins, je n'ai pas teste d'autre systeme). Installez le, tout est explique au sein de l'archive dans le fichier "INSTALL".

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

On retrouve ici nos entites insecables, la base du langage. On peut remarquer qu'il y en a beaucoup d'inutiles dans le cadre du projet minishell, on devra donc considerablement limiter ce vocabulaire.

Par exemple :

```lua
%token  WORD
%token  HEREDOC  DGREAT  LESS  GREAT
/*      '<<'      '>>'    '<'   '>'*/

%token  PIPE
/*      '|'  */
```

On ne garde que le token WORD et tous les operateurs dont nous avons besoin pour minishell. C'est bien plus court !

A la suite du fichier shell, nous avons une notation particuliere . EN effet, nous ne l'avons pas precise, mais le fichier shell est en fait un fichier nomenclature comme un fichier Yacc.

Ainsi, les regles syntaxiques sont presente sous la forme suivante :

```lua
cmd_name         : WORD 
							;
```
 Que concretement cela veut dire ? Que l'entite abstraite cmd_name (a gauche), est composee du token WORD. Par ailleurs, on peut remarquer que d'autre entites abstraites peuvent etre composees de plusieurs entites abstraite et/ou token. Exemple :
 
```lua
io_redirect      :           io_file
                 | IO_NUMBER io_file
                 |           io_here
                 | IO_NUMBER io_here
                 ;
```

Cela veut dire que l'entite abstraite io_redirect peut etre composee par :

- io_file
- IO_NUMBER io_file
- io_here
- IO_NUMBER io_here

Et ces differentes possibilites de compositions sont separees par le caractere '|', l'ensemble se terminant systematiquement pas ';'.

On voit donc que les entites abstraites sont des compositions. Et la raison en devient plus evidente a nesure qu'on comprend le fichier. Car en regarde bien, le debut de la partie concernant la syntaxe, nous avons ceci :

```lua
%start  complete_command
%%
complete_command : list separator
                 | list
                 ;
```

On commence par notre complete_command, et si on la decompose avec les parties a droite (ses definitions) et aue nous regardons par la suite en quoi sont definies ces parties et ainsi de suite, nous arriverions a un ou des tokens. Donc l'idee de cette grammaire, c'esrt de voir comment on peut arriver a l'entite "ultime" : une complete_command, c'est a dire une ligne de commande syntaxiquement correcte.

Cependantm encore une fois, cette grammaire est tres dense par rapport a ce dont nous avons besoin nous, et il faut donc tailler large. Ce travail est interessant a faire, car cela nous permet d'apprivoiser la syntaxe. Je ne mettrai pas de document contenant entierement ladite grammaire, d'autant que chacun eut faire a sa facon, changer les regles etc. Mais partir de celle du shell est une excellente idee et plus que recommandee dans tous les cas. N'hesitez pas a faire ce travail au sein d'un gichier ".y", cela nous sera utile pour la suite.

### Comprendre les implications de cette grammaire a l'aide de bison

Seulement, c'est bien beau tout ca, mais on a du mal a voir a quoi ca pourra bien servir, car nous ce qui nous interesse, c'est de coder. Mais avant tout, il nous reste a voir qu'est-ce que cette grammaire implique concretement.

Pour ce faire, une fois le fichier "ma_grammaire.y" de fait, passez le a bison :

`bison --html ma_grammaire.y`

Cela nous sortira un fichier html, que vous pouvez ouvrir avec firefox.

Ce fichier recapitule toutes nos regles, nous permet de voir s'il y a des conflits dans nos regles, etc. En bref, c'est un condense plutot lisible d'informations concernant notre syntaxe.

Et il y a egalement une partie qui nous sera precieuse, celle de l'automaton :

```lua
State 0

    0 $accept → • program $end
    1 program → • pipeline
    2         | • program AND_IF pipeline
    3         | • program OR_IF pipeline
    4 pipeline → • command

<SNIP>

  23 io_here → • HEREDOC WORD

    WORD     shift, and go to state 1
    HEREDOC  shift, and go to state 2
    DGREAT   shift, and go to state 3
<SNIP>
```

L'aumaton est un outil genial pour comprendre l'integralite des etapes a suivre pour un lexique donne.

Il nous donne des informations precieuses sur ce que nous pouvons faire dans tel ou tel cas.

### Shift Reduce

Mais avant toute chose, interessons-nous a deux mots : Shift et Reduce.

Cela est directement en rapport avec le type de parser que nous utilisons : un parser "LR" ([Left Right Parser](https://fr.wikipedia.org/wiki/Analyseur_LR "Left Right Parser"))

Nous n'allons pas rentrer dans les details, mais pour construire l"ast, nous avons besoin de deux choses essentielles :

- Une stack sur laquelle empiler une entite.
- Une tete de lecture sur les token afin de savoir ou nous en somme

Shift veut dire qu'on met telle entite sur la stack, nous conduisant a un autre etat.
Reduce veut qu'on reduit telle ou telle entite ou groupe d'entite selon l'une des regles de nos grammaire.

### Enchainement

Ainsi pour Shift, a l'etat 0 de mon fichier :

`WORD     shift, and go to state 1`

Veut dire qu'on met le token WORD sur la stack, qu'on avance la tete de lecture sur le buffer (on avance de 1 token, left -> right) et qu'on doit aller a l'etat 1 pour connaitre l'action suivante.

Qui est celle-cii :

```lua
State 1

   11 cmd_name → WORD •

    $default  reduce using rule 11 (cmd_name)
```
On voit qu'il n'y qu'une seule regle : le token WORD devient l'entite cmd_name (vous aurez peut-etre choisi un nom different). DOnc la stack est comme suit :

`STACK = cmd_name $`

Je reviens donc a mon etat initial. Sauf que la au lieu de considerer mon buffer, je maintenant considerer la stack, car elle n'est pas vide.

A l'etat 0, elle ne sera plus jamais vide des lors qu'on a lu le premier token, car l'idee, si la synatxe est bonne, sera d'avoir l'entite utile, et si ce n'est pas le cas, cela veut dire que la syntaxe est fausse (ou bien votre grammaire mais esperons que ce ne soit pas le cas ^^).

Par exemple, mon etat 0 c'est ca :

```lua
<SNIP>
   WORD     shift, and go to state 1
    HEREDOC  shift, and go to state 2
    DGREAT   shift, and go to state 3
    LESS     shift, and go to state 4
    GREAT    shift, and go to state 5

    program      go to state 6
    pipeline     go to state 7
    command      go to state 8
    cmd_name     go to state 9
    cmd_prefix   go to state 10
    io_redirect  go to state 11
    io_file      go to state 12
    io_here      go to state 13

```

Si je reprends mon exemple si dessus, j'ai maintenant "cmd_name" dans ma stack, et on me dit qu'il faut que j'aille a l'etat 9. Et ainsi de suite.

Pour donner un exemple, avec ma grammaire, pour la commade :

`ls -la`

J'ai la stack suivante (chronologiquement):

| Etape  | Stack  | Buffer  |
| ------------ | ------------ | ------------ |
|  0 | vide  | WORD WORD  |
| 1  | WORD  | WORD  |
| 2  | cmd_name  | WORD  |
|  3 | cmd_name WORD  | vide  |
|  4 | cmd_name cmd_suffix | vide  |
|  5 | simple_command  | vide  |
| 6  | pipeline  | vide  |
| 7  | program  | vide  |
| 8  | program end  | vide  |

Je ne detaille pas tout, mais ca vous donne une idee de la logique derriere.

### Mais que faire de cela ?

Eh bien... ce que vous voulez ! A l'aide de cet outil vous pouvez creer un arbre (avec une stack de noeuds), une liste chaine etc. Vous pouvez produire directement un arbre en vu de l'executions ou bien une liste decorelees de l'analyse syntaxique. Vous pouvez passer par le developpement d'une state machine pour singer ce fichier, vous faire une fichier de config, utiliser une recursion qui par de l'entite ultime pour arriver a un token, ou bien l'inverse, du token pour arriver a l'entite ultime (bottom up parser, la state machine fonctionne ainsi). Bref, las possibilite sont larges, et elle ne demandent qu'a etre developpees.

J'espere neanmoins que ce document vous permettra de vous faciliter le debut du chemin. Si des remarque etc. Discord : chbadad
