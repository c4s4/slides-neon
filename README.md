L'Outil de Build NeON
=====================

Michel Casabianca

casa@sweetohm.net

---

Proposition RMLL
----------------

Ayant eu l'occasion d'utiliser de nombreux systèmes de build (Make, Ant, Maven, et Rake), j'ai rêvé au fil des années le système de build idéal. En 2006, j'ai sauté le pas et développé Bee, mais les difficultés à la maintenir au fil des versions de Ruby m'ont contraint à en abandonner le support en 2014.

NeON est le successeur de Bee entièrement réécrit avec le langage Go. Il est:

- Rapide car Go est compilé en natif.
- Indépendant de la plateforme car porté sur 20 combinaisons OS/plateforme.
- Scriptable avec le langage de script Anko.
- Extensible par héritage de build.

Cette conférence présente NeON ainsi que des exemples de mise en œuvre en production.

Projet sur Github sous licence Apache 2.0 : http://github.com/c4s4/neon

---

Pourquoi NeON ?
---------------

Il existe bien des systèmes de build, pourquoi en concevoir un autre ? Parce que je n'étais pas satisfait de ceux que j'ai pu utiliser : **make**, **ant**, **maven**, **rake**.

En les utilisant et constatant leurs limites, j'ai rêvé d'un système de build qui aurait les caractéristiques suivantes :

- Indépendant du langage (non lié à *Java* comme peut l'être *Maven*).
- Indépendant du système sur lequel on build (comme *Make* est lié à *Unix*).
- Ayant des build files avec une syntaxe naturelle et légère (contrairement au *XML* de *Ant*).
- Rapide au lancement car on lance des builds des dizaines de fois par jour (pas comme *Maven*).
- Permettant le partage facile des build files (par des entrepôts *Git*).
- Permettant d'étendre des builds files parents par héritage (comme on le fait en *Programmation Orienté Objet*).
- Embarquant un langage de script pour être capable de coder des tâches complexes.

---

L'Ancêtre Bee
-------------

En 2006 je me suis lancé dans la conception de [Bee](https://github.com/c4s4/bee/), un outil de build qui répondait à cette liste de vœux. Il était implémenté en Ruby, le langage de script était donc Ruby lui même et la distribution des build files se faisait sous forme de *gemmes Ruby*.

Bee a été maintenu de 2006 à 2014 et a été utilisé dans nombre des projets chez OAB (une filière d'Orange pour laquelle j'ai travaillé 7 années).

J'ai finalement abandonné le projet suite à de grosses difficultés à la maintenir : il avait commencé sur Ruby *1.8.6* et chaque nouvelle version de Ruby obligeait à des adaptations dépendantes de la version.

D'autre part, l'installation de la machine virtuelle Ruby était un frein sérieux à son adoption.

---

Les débuts de NeON
------------------

La même année que j'arrêtais le développement de Bee, j'ai découvert le langage *Go*. J'y ai vite vu une solution aux problèmes de Bee : pas de VM à installer et pas de problèmes de soucis de maintenance au fil des versions de *Go*.

J'ai dû cependant attendre jusque fin *2016* avant de trouver une solution à deux problèmes auxquels je m'étais heurté lors de mes première tentatives de portage de Bee en *Go* :

- Trouver un langage de script embarqué dans du *Go*. J'ai trouvé pour [Anko](https://github.com/mattn/anko) qui est une version scriptée du *Go*.
- Pouvoir parser les build files en *Go*. Ils n'ont en effet pas une structure totalement prédéfinie et leur parsing nécessite la maîtrise de *l'introspection*, ce qui n'est pas une mince affaire en *Go* :o)

J'ai finalement commencé le développement fin 2016 et publié une première release début 2017.

---

Format des build files
----------------------

Les build files sont au format *YAML* (pour *Yet Another Markup Language*). Ce format a l'avantage d'être naturel et léger. Par exemple, on écrira une liste de la manière suivante :

```yaml
- Bilbo
- Frodo
- Gandalf
```

Et un dictionnaire comme suit :

```yaml
gandalf: istari
bilbo: hobbit
frodo: hobbit
galadriel: elfe
```

Il existe aussi une forme compacte :

```yaml
list: [Bilbo, Frodo]
map:  {good: Frodo, bad: Sauron}
```

---

### Types de données

D'autre part, les données sont typées :

```yaml
integers:
  - 1
  - 123
floats:
  - 1.2
  - 3.14
dates:
  - 2015-10-21
strings:
  - This is text
  - "1"
  - '2015-10-21'
```

Ainsi les nombres sont écrits comme des nombres, les dates sont écrites au format ISO, les booléens sont *true* et *false* et tout le reste sont des chaînes de caractères. On peut forcer le type *chaîne de caractères* en entourant de guillemets. Ainsi `"123"` sera une chaînes de caractères et non un nombre.

---

### Conseils YAML

Pour éviter tout problème lors de l'écriture de build files (et de fichiers YAML plus généralement) :

- On ne doit **pas indenter avec des tabulations** (c'est une faute de syntaxe en YAML).
- Les chaînes de caractères comportant un caractère *deux points* doivent être entourées de guillemets, sans qui elles sont considérées comme un dictionnaire.

YAML est un format très semblable à JSON à tel point que JSON est un sous-ensemble de YAML (dans la spécification de YAML, pas dans celle de JSON :o)

Bien sûr la syntaxe de YAML est bien plus riche que celle décrite ici, et je vous invite à l'approfondir avec mon [Introduction à YAML](http://sweetohm.net/article/introduction-yaml.html) ou sur le [site officiel de YAML](http://yaml.org/) qui comprend la spécification.

---

Structure des Build Files
-------------------------

Un build file est un dictionnaire YAML pouvant comporter les entrées suivantes :

- **doc** pour documenter le build.
- **default** indique la cible par défaut du build.
- **extends** indique les build files parents.
- **properties** définit les propriétés du build.
- **environment** définit les variables d'environnement du build.
- **targets** comporte les cibles du build (comme *clean* ou *compile* par exemple).

Il existe d'autres entrées de moindre importance, comme **repository**, **context**, **singleton**, **shell**, **version** que je ne détaillerai pas dans ces slides. Je vous invite à consulter le [Guilde de l'utilisateur (en anglais)](https://github.com/c4s4/neon/blob/master/doc/usermanual.md) pour plus de précisions.

---

### Exemple de Build File

```yaml
default: test

properties:
  NAME:      'mytool'
  BUILD_DIR: 'build'

targets:

  test:
    doc: Run Go tests
    steps:
    - $: ['go', 'test']

  bin:
    doc: Build Go tool
    steps:
    - mkdir: '=BUILD_DIR'
    - $: ['go', 'build', '-o', '={BUILD_DIR}/={NAME}']

  clean:
    doc: Clean generated files
    steps:
    - delete: '=BUILD_DIR'
```

---

### Les Propriétés du Build

Les propriétés du build sont l'équivalent des variables d'un langage de programmation. Elles sont définies dans l'entrée *properties* du build file. On peut définir comme propriété tous les types YAML :

```yaml
properties:
  STRING:  'This is a string'
  OTHER:   '1'
  INTEGER: 42
  FLOAT:   4.2
  LIST:    [1, 1, 2, 3, 5, 8]
  MAP:
    one:   1
    two:   2
    three: 3
```

Une propriété peut faire référence à une autre propriété :

```yaml
properties:
  NAME:      'test'
  BUILD_DIR: 'build'
  ARCHIVE:   '={BUILD_DIR}/={NAME}.zip'
```

---

### Références à des propriétés

Il y a deux manières de référencer une propriété :

- En tant que chaîne de caractères sous la forme `={PROPRIETE}`.
- En tant que valeur ayant le type de la propriété sous la forme `=PROPRIETE`.

Ainsi, le build file suivant :

```yaml
properties:
  LIST: ['foo', 'bar']

targets:

  test:
    steps:
    - print: 'LIST: ={LIST}'
    - for: e
      in:  =LIST
      do:
      - print: =e
```

Affiche la liste puis chacun de ses éléments.

---

### Propriétés prédéfinies

NeON définit des propriétés par défaut :

```yaml
targets:

  test:
    steps:
    - print: 'BASE: ={_BASE}'
    - print: 'HERE: ={_HERE}'
    - print: 'OS:   ={_OS}'
    - print: 'ARCH: ={_ARCH}'
    - print: 'NCPU: ={_NCPU}'
```

Produit la sortie suivante :

```bash
$ neon test
----------------------------------------------------------------------- test --
BASE: /home/casa/dsk
HERE: /home/casa/dsk
OS:   linux
ARCH: amd64
NCPU: 2
OK
```

---

Build Targets
-------------

Les cibles du build sont comparables à des fonctions. On les passe sur la ligne de commande. Ainsi avec le build file suivant :

```yaml
properties:
  BUILD_DIR: 'build'

targets:

  clean:
    doc: Clean generated files
    steps:
    - delete: =BUILD_DIR
```

Si on tape `neon clean`:

```bash
$ neon clean
Build /home/casa/dsk/build.yml
----------------------------------------------------------------------- clean --
OK
```

Nous voyons que la cible *clean* a été exécutée.

---

Il est aussi possible d'exécuter plusieurs cibles en les passant sur la ligne de commande.

D'autre part, on peut déclarer une cible par défaut avec l'entrée `default: target` à la racine du build file, comme dans l'exemple suivant qui déclare la cible *clean* par défaut:

```yaml
default: clean

properties:
  BUILD_DIR: 'build'

targets:

  clean:
    doc: Clean generated files
    steps:
    - delete: =BUILD_DIR
```

On peut donc invoquer NeON sans passer de cible, comme suit :

```bash
$ neon
Build /home/casa/dsk/build.yml
----------------------------------------------------------------------- clean --
OK
```

---

