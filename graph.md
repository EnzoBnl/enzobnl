<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->

# Théorie des Graphes
## Définitions
### Permutations
*Nombre de permutations de $k$ éléments*.

On a $n$ éléments et $n$ slots **ordonnés** où les placers :

`12345...n`

`_____...n`

On a $n$ possibilités pour placer le premier, puis $n-1$ pour le second $......$ puis une seule possibilité pour le $n^{ème}$.

On a donc $n!$ permutations de nos $n$ éléments.
### Arrangements
*Nombre d'arrangements de $k$ éléments dans un ensemble de taille $n$.*

Soit n éléments $12345......n$.

On veut en choisir $k$ sans remise.

Pour le premier on a le choix entre $n$ éléments, pour le second entre $n-1$ éléments$......$ pour le $k^{ème}$ 
entre $n-k+1$ éléments.

Le nombre de scenarios de succession de $k$ choix d'éléments parmi $n$ =  $n(n-1)(n-2)......(n-k+1)=\frac{n!}{(n-k)!}$

### Coefficient binomial
*Nombre de sous-ensembles de taille $k$ dans un ensemble de taille $n$.*

Soit n éléments $12345......n$.

On veut en choisir $k$ sans remise **ni distinction d'ordre**.

On débute avec le même raisonnement que pour l'arrangement sauf que l'ordre des choix n'importe pas. Nous nous retrouvons avec beaucoup d'ensemble identiques car nous avons pour chaque ensemble toutes ses permutations possibles : il faut diviser le précédent résultat par le nombre d'arrangements d'un ensemble de taille $k$, soit $k!$

On obtient donc ${n \choose k}=\frac{n!}{k!(n-k)!}=\frac{\vert arrangements(n, k)\vert }{\vert permutations(k)\vert }$


### Chaine et chemin
1. Dans un graphe non-orienté : Une **chaine** $\mu (x,y)$ reliant $x$ et $y$ est une suite finie d'arêtes consécutives reliant $x$ et $y$.
2. Dans un graphe orienté : Un **chemin** $\mu [x,y]$ d'origine $x$ et d'extrémité $y$ est une suite finie d'arcs consécutifs reliant $x$ à $y$.

Une chaine simple $\mu (x,y)$ est un **cycle**  $\Leftrightarrow x=y$

Un chemin $\mu [x,y]$ est un **circuit**  $\Leftrightarrow x=y$

Une chaine (ou un chemin) :
- est **élémentaire** si tous ses sommets sont distincts.
- est **simple** si toutes ses arêtes (resp. ses arcs) sont disctincts

|FR|EN|
|-:|:-|
| chaine  | walk |
| chaine simple | trail |
| chaine élémentaire | path |
| chemin | directed walk |
| chemin simple | directed trail |
| chemin élémentaire | directed path, dipath|


### Connexité
- Un graphe $G=(V,E)$ non-orienté (*resp. orienté*) est connexe  $\Leftrightarrow \exists v\in V(G),\forall v'\in V(G),\exists$ une **chaine** (*resp. chemin*) reliant $v$ et $v'$
- Un graphe $G=(V,E)$ orienté est **fortement** connexe  $\Leftrightarrow \forall v,v'\in V(G),\exists$ un **chemin** allant de $v$ vers $v'$

### Séparateur (Vertex separator)
Soit $G = (V, E)$ un graphe. 

$S\subset V$ est un séparateur de $G$ si :

$\exists (u,v)\in (V\setminus S)^2$ tel que $u$ et $v$ sont reliés dans $G$ mais ne le sont plus dans  $G' = (V\setminus S, E)$.
### Isthme (bridge)
Soit $G = (V, E)$ un graphe. 

$i\in E$ est un isthme de $G$ si :

$\exists (u,v)\in V^2$ tel que $u$ et $v$ sont reliés dans $G$ mais ne le sont plus dans  $G' = (V, E\setminus \{i\})$.
### Degré
Le degré d'un sommets $s$ est le nombre d'arêtes impliquant $s$, les boucles comptant deux fois.

Si le graphe est orienté, c'est la somme du nombre d'arêtes entrantes et du nombre d'arêtes sortantes.
### Arbre (=arbre enraciné)
Graphe **acyclique orienté** $G=(V,E)$, possédant un sommet racine $r\in V$ tel que $deg_{in}(r) = 0$ et $\forall v \in V\setminus \{r\}, deg_{in}(v) = 1$

- Si $deg_{out}(v) = 0$, $v$ est une **feuille**, sinon c'est un **noeud**.
- Si $\forall v \in V, deg_{out}(v) \leq 2$, l'arbre est dit **binaire**.

## Theorèmes
### Chaîne/Chemin $\implies$ Chaine/Chemin élémentaire
Soit $G=(V, E)$ un graphe non orienté (*resp. orienté*).

Soit $a,b\in V$, $\exists w$ une chaine (*resp. chemin*) de $a$ vers $b \implies \exists p$, une chaine (*resp. chemin*) élémentaire de $a$ vers $b$.
### Transitivité des chemins

Soit $x,y,z \in G$.

Il existe un chaine (ou chemin) élémentaire de $x$, vers $y$ et un chaine (ou chemin) élémentaire de $y$, vers $z$ $\implies$ Il existe un chaine (ou chemin) élémentaire de $x$, vers $z$
### Handshake Theorem
$\forall G=(V,E),\space \sum\limits_{v \in V(G)}deg(v)=2|E|$

### Sommets de degré impair
$\forall G=(V,E),\space| $$\{$$v \in V,\space deg(v)\space est \space impair$$\}$$|\space est \space pair$


Preuve   : ${(1)}$
### Chemin dans un arbre
$T=(V,E)$ est un arbre $\implies \forall a,b\in V$,  il existe un unique chemin $\mu$ reliant $a$ à $b$ ou $b$ à $a$.
### Arbre et isthme
Toute arête d'un arbre est un **isthme**.
### Nombre d'arête dans un arbre
$T=(V,E)$ est un arbre $\implies |E|=|V|-1$
## Vers le PageRank
### Variable aléatoire
C'est une application définie sur l'ensemble des éventualités (=l'ensemble des résultats possibles d'une expérience aléatoire)
### Processus stochastique (=processus aléatoire)
C'est une évolution, à pas discret ou continu, d'une variable aléatoire.
### Processus de Markov
C'est un processus stochastique possédant la *propriété de Markov* : La prédiction du futur du processus à partir de son présent n'est pas rendue plus précise par la connaissance additionnelle de son passé.
### Chaine de Markov
C'est une réalisation d'un processus de Markov à temps discret ou continu et à espace d'état discret noté $E$: 

C'est une séquence $X_0, X_1, X_2, ...$ de variables aléatoires $X_n \in E$ étant l'état du processus à l'instant $n$.
- Propriété d'ergodicité: Les propriétés statistiques du processus sont estimables à partir d'un échantillon suffisamment grand.

### Matrice de transition d'une chaine de Markov
$$
M=\begin{pmatrix}  
a_{1,1} & a_{1,2} \\  
a_{2,1} & a_{2,2}
\end{pmatrix}
$$

Sachant que $i$ est le dernier état d'une chaine de Markov, $a_{i,j}$ est la probabilité que le suivant soit $j$.

La matrice de transition de la chaine est égale à la *matrice d'adjacence* du graphe de la chaine de Markov :

<div class="mermaid">
graph TB
1[1] --a<sub>1,1</sub>--> 1
2[2] --a<sub>2,2</sub>--> 2
2 --a<sub>2,1</sub>--> 1
1 --a<sub>1,2</sub>--> 2
</div>
  
Ce graphe ignore les arêtes qui aurait une probabilité nulle. Il n'est donc pas forcément connexe.
### Probabilité stationnaire d'une Chaine de Markov
Soit $X=(X_n)_n \geq 0$ une chaine de Markov.

Sa probabilité stationnaire $\pi _ i =lim _ {n \rightarrow \inf} \frac{1 _ {X _ 0=i}+1 _ {X _ 1=i}+...+1 _ {X _ {n-1}=i}}{n}=lim _ {n \rightarrow \inf} \frac{S _ n(i)}{n}$. C'est la part du temps que le processus passe dans l'état $i$.
### Théorème de Perron Frobenius
...
### PageRank
En considérant $G=(V,E)$ avec $V=pages, E=hyperliens \space pondérés$, **le PageRank d'un élément de** $V$ **est la probabilité stationnaire correspondante de la chaine de Markov associée à** $G$, c'est-à-dire un vecteur de Perron-Frobenius de la matrice d'adjacence du graphe du Web.

**La solution analytique étant impossible à calculer** du fait de la taille du graphe et de son évolution (modifications de pages et hyperliens, connexion ou déconnexion de serveur web) un **algorithme d'approximation** est utilisé:

C'est un algo itératif dont la terminaison est assurée par un nombre d'itérations fixé à l'avance ou par l'utilisation d'un seuil $\epsilon$ de variation qui garantie la terminaison grâce au théorème de **convergence des probabilités stationnaires d'une chaine de Markov**.

Preuve   : ${(2)}$

## Graph of the Web, basic web properties

### Loi de puissance pour le degré entrant
Soit $P_{deg-in}(i)$ probabilité qu'un noeud ait un degré entrant $i$.

Soit $x>1$

$P_{deg-in}(i)\propto \frac{1}{i^x}$

### Taille du web
- en 2008, 1.3 trilliard de pages dont 1 trilliard dans web invisible (= web non référencé = deep web)

|nom|def|
|--|--|
|web|toutes les pages web accessibles|
|web visible|pages indexées par les moteurs de recherche|
|web opaque|pages indexables mais non indexées par les moteurs de recherche|
|deep web|pages non indexées|
|dark web| pages non indexées et accessibles anonymement via serveur et couches d'encryption|

|si|$10^{...}$|naming|
|--|--|:--:|
|kilo| $10^{3}$ | mille |
|mega| $10^{6}$ | million |
|giga| $10^{9}$ | milliard |
|tera| $10^{12}$ | billion = mille milliards |
|peta| $10^{15}$| billiard = million de millards|
|exa| $10^{18}$ | trillion = milliard de milliards|
|zetta| $10^{21}$ | trilliard = mille milliards de milliards|
|yotta| $10^{24}$ | quadrillion = million de milliards de milliards|
|...| ... | ... |



## Preuves
### (1) : 
***Lemme***: Si ce théorème est vrai pour les graphes connexes, alors il est un chaine (ou chemin) élémentaire de $y$, vers $z$ $\implies$ Il existe un chaine (ou chemin) élémentavrai pour tous les graphes puisqu'une somme de nombres pairs est paires.

**Preuve par récurrence pour les graphes connexes:**
*Initialisation:*
Cas du graphe vide : $G_{Ø}=(Ø, Ø)$ possède 0 sommets de degré impair, le théorème est vérifié.

Soit $G=(\{a\}, Ø)$. $G$ vérifie le théorème puisqu'il possède 0 sommets de degré impair.

*Récurrence :*

Toute les topologies de graphes connexes sont constructibles à partire de $G$ en ajoutant une arête vers un sommets existant ou vers un sommet à créer.

Soit $imp(G) =\vert\{v\in V(G),\space \deg(v)\space impair\}\vert$ 

Étapes de construction possibles pour passer d'un graphe connexe $G_n$ vérifiant "$imp(G_n)$ est pair"  à $G_{n+1}$:

- Ajouter une arête depuis $v_1 \in V(G)$ vers $v_2\in V(G)$ ($v_1 \neq v_2$):
  - Si $deg(v_1)$ et $deg(v_2)$ sont pairs alors $imp(G_{n+1})$ = $imp(G_{n})+1+1$ est aussi pair.
  - Si $deg(v_1)$ et $deg(v_2)$ sont impairs alors $imp(G_{n+1})$ = $imp(G_{n})-1-1$ est aussi pair.
  - Si $deg(v_1)$ et $deg(v_2)$ sont de parité opposée alors $imp(G_{n+1})$ = $imp(G_{n})+1-1$ est aussi pair.
- Ajouter une arête depuis $v_1 \in V(G)$ vers $v_1$ alors la parité de $deg(v_1)$ ne change pas puisqu'on lui ajoute $+2$ par définition deu degré impair est pair.***
Preuve   : ${(1)}$.
-  Ajouter une arête depuis $v_1 \in V(G)$ vers $v_2\notin V(G)$ :
   - Si $deg(v_1)$ est pair alors $imp(G_{n+1})$ = $imp(G_{n})+1+1$ est aussi pair.
   - Si $deg(v_1)$ est impair alors $imp(G_{n+1})$ = $imp(G_{n})-1+1$ est aussi pair.
 
Donc par récurrence tout graphe connexe vérifie le théorème et par le *Lemme* précédent, ce résultat s'étend à tous les graphes.
