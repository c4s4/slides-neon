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
- Indépendant de la plateforme et porté sur 20 combinaisons OS/plateforme.
- Scriptable avec le langage Anko (<http://github.com/mattn/anko>).
- Extensible par héritage de build.

Cette conférence présente NeON ainsi que des exemples de mise en œuvre en production.

Projet sur Github sous licence Apache 2.0 : http://github.com/c4s4/neon

---

Pourquoi NeON ?
---------------

Il existe bien des systèmes de build, pourquoi en concevoir un ènième ? Parce que je n'étais pas satisfait de ceux que j'ai pu utiliser : **make**, **ant**, **maven**, **rake**.

En les utilisant et constatant leurs limites, j'ai rêvé d'un système de build qui aurait les caractéristiques suivantes :

- Indépendant du langage (non lié à *Java* comme peut l'être *Maven*).
- Indépendant du système sur lequel on build (comme *Make* est lié à *Unix*).
- Ayant des fichiers de build avec une syntaxe naturelle et légère (contrairement au *XML* de *Ant*).
- Rapide au lancement car on lance des builds des dizaines de fois par jour (pas comme *Maven*).
- Permettant le partage facile des fichiers de build (par des entrepôts *Git*).
- Permettant d'étendre des builds files parents par héritage (comme on le fait en *Programmation Orienté Objet*).
- Embarquant un langage de script pour être capable de coder des tâches complexes.

---

L'Ancêtre Bee
-------------

En 2006 je me suis lancé dans la conception de [Bee](https://github.com/c4s4/bee/), un outil de build qui répondait à cette liste de vœux. Il était implémenté en Ruby, le langage de script était donc Ruby lui même et la distribution des fichiers de build se faisait sous forme de *gemmes Ruby*.

Bee a été maintenu de 2006 à 2014 et a été utilisé dans nombre des projets chez OAB (une filière d'Orange pour laquelle j'ai travaillé 7 années) et ailleurs.

J'ai finalement abandonné le projet suite à de grosses difficultés à le maintenir : il avait commencé sur Ruby *1.8.6* et chaque nouvelle version de Ruby obligeait à des adaptations dépendantes de la version.

D'autre part, l'installation de la machine virtuelle Ruby était un frein sérieux à son adoption.

---

Les débuts de NeON
------------------

La même année que j'arrêtais le développement de Bee, j'ai découvert le langage de programmation *Go*. J'y ai vite vu une solution aux problèmes de Bee : pas de VM à installer et pas de soucis de maintenance au fil des versions de *Go*.

J'ai dû cependant attendre jusque fin *2016* avant de trouver une solution à deux problèmes auxquels je m'étais heurté lors de mes première tentatives de portage de Bee en *Go* :

- Trouver un langage de script embarqué dans du *Go*. J'ai opté pour [Anko](https://github.com/mattn/anko) qui est une version scriptée du *Go*.
- Pouvoir parser les fichiers de build en *Go*. Ils n'ont en effet pas une structure totalement prédéfinie et leur parsing nécessite la maîtrise de *l'introspection*, ce qui n'est pas une mince affaire en *Go* :o)

J'ai finalement commencé le développement fin 2016 et publié une première release début 2017.

---

Format des fichiers de build
----------------------------

Les fichiers de build sont au format *YAML* (pour *YAML Ain't Markup Language*). Ce format a l'avantage d'être naturel et léger. Par exemple, on écrira une liste de la manière suivante :

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

Il existe aussi une forme compacte pour les listes et les dictionnaires :

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

Ainsi les nombres sont écrits comme des nombres, les dates sont écrites au format ISO, les booléens sont *true* et *false* et tout le reste sont des chaînes de caractères. On peut forcer le type *chaîne de caractères* en entourant de guillemets (simples ou doubles). Ainsi `"123"` sera une chaînes de caractères et non un nombre.

---

### Conseils YAML

Pour éviter tout problème lors de l'écriture de fichiers de build (et de fichiers YAML plus généralement) :

- On ne doit **pas indenter avec des tabulations** (c'est une faute de syntaxe en YAML).
- Les chaînes de caractères comportant un caractère *deux points* doivent être entourées de guillemets, sans quoi elles sont considérées comme un dictionnaire.

YAML est un format très semblable à JSON à tel point que JSON est un sous-ensemble de YAML (dans la spécification de YAML, pas dans celle de JSON :o)

Bien sûr la syntaxe de YAML est bien plus riche que celle décrite ici, et je vous invite à l'approfondir avec mon [Introduction à YAML](http://sweetohm.net/article/introduction-yaml.html) ou sur le [site officiel de YAML](http://yaml.org/) qui comprend la spécification.

---

Structure des fichiers de build
-------------------------------

Un fichier de build est un dictionnaire YAML pouvant comporter les entrées suivantes :

- **doc** pour documenter le build.
- **default** indique la cible par défaut du build.
- **extends** indique les fichiers de build parents.
- **properties** définit les propriétés du build.
- **environment** définit les variables d'environnement du build.
- **targets** comporte les cibles du build (comme *clean* ou *compile* par exemple).

Il existe d'autres entrées de moindre importance, comme **repository**, **context**, **singleton**, **shell**, **version** que je ne détaillerai pas dans ces slides. Je vous invite à consulter le [Guilde de l'utilisateur (en anglais)](https://github.com/c4s4/neon/blob/master/doc/usermanual.md) pour plus de précisions.

---

### Exemple de fichier de build

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
    - mkdir: =BUILD_DIR
    - $: ['go', 'build', '-o', '={BUILD_DIR}/={NAME}']

  clean:
    doc: Clean generated files
    steps:
    - delete: =BUILD_DIR
```

---

### Les Propriétés du Build

Les propriétés du build sont l'équivalent des variables d'un langage de programmation. Elles sont définies dans l'entrée *properties* du fichier de build. On peut définir comme propriété tous les types YAML :

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

Affiche la liste puis chacun de ses éléments :

```bash
LIST: [foo, bar]
foo
bar
```

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

Cibles du fichier de build
--------------------------

Les cibles du build sont comparables à des fonctions. On les passe sur la ligne de commande. Ainsi avec le fichier de build suivant :

```yaml
properties:
  BUILD_DIR: 'build'

targets:

  clean:
    doc: Clean generated files
    steps:
    - delete: =BUILD_DIR
```

On exécutera la cible *clean* avec la commande `neon clean`:

```bash
$ neon clean
Build /home/casa/dsk/build.yml
----------------------------------------------------------------------- clean --
Deleting 1 file(s) or directory(ies)
OK
```

---

Il est aussi possible d'exécuter plusieurs cibles en les passant sur la ligne de commande.

D'autre part, on peut déclarer une cible par défaut avec l'entrée `default: target` à la racine du fichier de build, on peut alors invoquer NeON sans passer de cible :

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

La cible d'un fichier de build comporte les champs suivants :

- **doc** permet de documenter la cible et est affiché avec l'option *-info*.
- **depends** indique les cibles à exécuter auparavant.
- **steps** liste les étapes de la cible (ou les *tâches* qui la constituent).

---

Les tâches sont comparables aux instructions d'un programme. Elles peuvent être de trois types :

### Tâches NeON

Ces tâches sont prédéfinies dans le moteur NeON, elles permettent de **gérer les fichiers** (copie, effacement, déplacement), les **archives** (créer et décompresser des fichiers *tar*, *targz* et *zip*), les **répertoires** (création, effacement) ou des **liens**. Ceci permet de réaliser ces opérations de manière indépendante du système d'exploitation.

Par exemple, pour effacer tous les fichiers *.so* du répertoire *build*, on pourrait écrire :

```yaml
targets:

  delete:
    doc: Delete object files
    steps:
    - delete: '**/*.so'
      dir:    'build'
```

---

#### Tâches NeON (suite)

Il existe aussi des tâches logiques qui permettent de réaliser des **tests** ou d'**itérer** sur des collections. Par exemple, pour itérer sur tous les fichiers *.md* du répertoire *md*, on peut écrire :

```yaml
- for: 'file'
  in:  'find("md", "*.md")'
  do:
  - $: ['md2pdf', '-o', 'build/={file}.pdf', 'md/={file}']
```

Il est aussi possible de gérer les erreurs. Par exemple, pour exécuter une commande et récupérer la main en cas d'erreur, on peut écrire :

```yaml
- try:
  - $: 'command that might fail'
  catch:
  - throw: 'There was an error running command'
```

Ceci permet de corriger des erreurs ou de produire des messages plus explicites.

---

#### Tâches NeON (suite)

Pour lister, en ligne de commande, toutes les tâches NeON disponibles, on peut taper `neon -tasks`. Pour obtenir de l'aide sur une tâche particulière, on tapera :

```bash
$ neon -task time
Record duration to run a block of steps.

Arguments:

- time: steps we want to measure execution duration (steps).
- to: property to store duration in seconds as a float, if not set, duration is
  printed on the console (string, optional).

Examples:

    # print duration to say hello
    - time:
      - print: 'Hello World!'
      to: duration
    - print: 'duration: ={duration}s'
```

On peut aussi obtenir de l'aide pour toutes les tâches disponibles [sur la page de référence des tâches](https://github.com/c4s4/neon/blob/master/doc/tasks.md).

---

#### Les tâches de fichiers

Nombre de tâches NeON travaillent sur des fichiers. Par exemple, la tâche *copy* :

```yaml
- copy:  ['**/*.txt', '**/*.md']
  dir:   'txt'
  todir: 'dst'
  flat:  true
```

Le champ *copy* définit la liste des fichiers à copier avec des *globs*, qui sont semblables à ceux utilisés en ligne de commande :

- **\*** pour sélectionner un nombre quelconque de caractères.
- **?** pour sélectionner un caractère. Donc `?.txt` sélectionnera *1.txt* mais pas *12.txt*.
- **\*\*** pour sélectionner un nombre quelconque de répertoires et sous-répertoires. Donc `**/*.txt` sélectionnera *foo.txt*, *foo/bar.txt* et tous les fichiers *.txt* dans les sous-répertoires.

Le champ **dir** indique le répertoire racine des globs, **exclude** liste les globs des fichiers à exclure, **todir** ou **tofile** indiquent la destination de la copie et **flat** indique si la copie ramène les fichiers à la racine du répertoire de destination.

---

### Les tâches shell

Les tâches *shell* exécutent des commandes système. Par exemple, pour exécuter la commande *ls*, on pourrait écrire :

```yaml
- $: 'ls'
```

Pour exécuter des commandes qui soient compatibles avec toutes les plateformes, il est recommandé d'écrire ces tâches sous forme de listes :

```yaml
- $: ['java', '-jar', 'my.jar', 'Hello World!']
```

Ainsi on évite d'invoquer les commandes en passant par *cmd.exe* sous Windows, qui gère très mal les arguments comportant des espaces. Il est ainsi possible d'écrire des tâches *shell* qui sont portables entre plateformes.

C'est le cas pour les commande Git par exemple. On peut ainsi écrire des fichiers de build qui comportent des commandes système *et* sont portables entre plateformes (*Unix* et *Windows* en particulier).

---

### Les tâches script

NeON embarque une VM Anko qui permet d'écrire des scripts dans les fichiers de build. Anko est un langage de script très proche du *Go* et permet de réaliser des tâches complexes indépendantes de la plateforme. D'autre part, NeON définit des fonctions utiles pour un build, les *builtins*.

Par exemple, on peut rechercher des fichiers avec *find* et les filter avec *filter*. Ainsi pour sélectionner tous les fichiers *txt* saufs ceux du répertoire *build*, on pourra écrire :

```yaml
- 'files = filter(find(".", "**/*.txt"), "build/**/*")'
```

La propriété *files* contiendra une liste des fichiers trouvés.

On peut lister tous les builtins avec la commande `neon -builtins` et obtenir de l'aide sur l'un d'eux avec `neon -builtin find` par exemple.

Il est possible de définir ses propres builtins dans un source Anko et de les charger avec une déclaration *context*. Par exemple, pour charger le fichier *myscript.ank* dans le contexte du build, on écrira `context: myscript.ank`.

---

Héritage de build
-----------------

On peut étendre un fichier de build en ajoutant une déclaration extends.

```yaml
properties:
  BUILD_DIR: 'build'

targets:

  clean:
    doc: Clean generated files
    steps:
    - delete: '=BUILD_DIR'
```

Nous pouvons réutiliser ce fichier de build dans un autre de la manière suivante :

```yaml
extends: ./buildir.yml
```

Dans ce fichier de build, le propriété *BUILD_DIR* est définie et la cible *clean* peut être invoquée. Nous pouvons ainsi réutiliser les fichiers de build existant.

---

### Surcharge

Nous pouvons aussi surcharger dans le fils les propriétés définies dans le fichier de build parent. Ainsi pour changer le répertoire du build, nous pouvons écrire :

```yaml
properties:
  BUILD_DIR: 'target'
```

Nous pouvons aussi surcharger les cibles :

```yaml
extends: ./buildir.yml

targets:

  clean:
    doc: Clean generated files and create build directory
    steps:
    - super:
    - mkdir: =BUILD_DIR
```

La tâche `super` permet ainsi d'invoquer la cible du parent.

---

Entrepôt NeON
-------------

L'entrepôt est l'endroit où se trouvent généralement les fichiers de build parents et les templates NeON. Par défaut il est dans le répertoire *~/.neon/*. Un plugin est un projet Github identifié par le nom du compte et celui du projet.

Ainsi mon plugin pour mes fichiers de build parents se trouve à l'adresse <http://github.com/c4s4/build> et son nom NeON est donc *c4s4/build*. On fera donc référence à un fichier de build parent par `extends: c4s4/build/golang.yml` par exemple.

Pour installer ce plugin, on tapera sur la ligne de commande `neon -install c4s4/build`. NeON va alors cloner le projet Github se trouvant à l'adresse <http://github.com/c4s4/build> dans le répertoire *~/.neon/c4s4/build*.

Par défaut, on clonera la branche principale du plugin, ce qui est souvent ce que l'on veut. On peut cependant changer de branche par une commande Git. Pour passer sur *develop*, on tapera par exemple `git checkout develop` dans le répertoire du plugin. On peut aussi sortir un tag particulier, avec `git checkout 1.2.3` par exemple.

---

Templates NeON
--------------

Un plugin peut aussi comporter des templates. Ce sont des moyens de créer des projets rapidement. Par exemple, pour créer de nouveaux slides, je tape :

```bash
$ neon -template slides
Build /home/casa/.neon/c4s4/build/slides.tpl
-------------------------------------------------------------------- template --
This template will generate a Slides project
Name of this project: test
Making directory '/home/casa/dsk/test'
Copying 17 file(s)
Replacing text in file '/home/casa/dsk/test/build.yml'
Project generated in 'test' directory
OK
```

Ceci a créé un répertoire *test*, du nom du projet que j'ai saisi en ligne de commande. Ce répertoire contient un nouveau projet de slides, avec fichiers d'exemple et de build.

```bash
$ ls
build.yml  CHANGELOG.yml  img  README.md  res
```

---

### Fichiers de template

On peut créer ses propres templates. Ce sont des fichiers de build standards avec l'extension *.tpl*. Par exemple, voici les sources du template des slides :

```yaml
# prompt project name, create directory and copy files
- print: 'This template will generate a Slides project'
- prompt:  'Name of this project'
  to:      'name'
  pattern: '^[\w-_]+$'
  error:   'Project name must be made of letters, numbers, - and _'
- if: 'exists(joinpath(_HERE, name))'
  then:
  - throw: 'Project directory already exists'
- mkdir: '={_HERE}/={name}'
- copy:  '**/*'
  dir:   '={_BASE}/slides'
  todir: '={_HERE}/={name}'
# rename project in build file
- replace: '={_HERE}/={name}/build.yml'
  with:    {"'Slides'": =name}
- print: "Project generated in '={name}' directory"
```

---

### Fonctionnalités avancées

#### Singleton

Il est possible de s'assurer qu'une seule instance de build tourne sur une machine avec une entrée *singleton* à la racine du fichier de build. Elle comporte un numéro de port, qui sera ouvert en lecture en début de build et fermée lorsque le build est terminé. A noter que le numéro de port doit être supérieur à 1024 si on ne lance pas le build en *root*.

```yaml
singleton: 12345
```

C'est utile lorsque le build utilise une ressource qui ne peut être partagée. Si on lançait plusieurs builds en parallèle, cela conduirait alors à une erreur. Si on lance une deuxième instance, on a le message d'erreur suivant :

```bash
$ neon
Build /home/casa/dsk/build.yml
ERROR listening singleton port: listen tcp :12345: bind: address already in use
```

---

#### Builds parallèles

Nos machines actuelles comportent souvent plusieurs cœurs par processeur. Or NeON ne s'exécute par défaut que sur un seul d'entre eux, de manière séquentielle. Cependant, il arrive parfois que l'on souhaite effectuer plusieurs tâches en parallèle.

C'est possible avec la tâche *threads*. Considérons l'exemple ci-dessous :

```yaml
# compute squares of 10 first integers in threads and put them in _output
- threads: =_NCPU
  input:   =range(10)
  steps:
  - '_output = _input * _input'
  - print: '#{_input}^2 = #{_output}'
# print squares on the console
- print: '#{_output}'
```

Nous lançons ici en parallèle autant de threads qu'il y a de cœurs dans le CPU : `threads: =_NCPU`.

En entrée, nous prenons la liste des 10 premiers entiers : `input: =range(10)`. Cette liste sera consommée par l'ensemble des threads.

---

#### Builds parallèles (suite)

Viennent ensuite les étapes exécutées par chaque thread :

```yaml
steps:
- '_output = _input * _input'
- print: '#{_input}^2 = #{_output}'
```

Nous calculons la *sortie* du thread qui est la carré de son *entrée*. Les valeurs envoyées par chaque thread dans *_output* se retrouvera (dans un ordre indéterminé) dans l'*_output* après la tâche *threads*.

Si nous exécutons ce build, nous obtenons, per exemple :

```bash
$ n
Build /home/casa/dsk/build.yml
------------------------------------------------------------------------ test --
0^2 = 0
1^2 = 1
...
9^2 = 81
[0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
OK
```

---

#### Builds parallèles (suite)

Chaque thread obtient du build une **copie** du contexte du build (avec ses propriétés). Il peut modifier à sa guise ce contexte, cela n'affectera pas les autres threads qui ont leur propre contexte. Cependant, toutes les modifications au contexte d'un thread **sont perdues**, sauf ce qui a été envoyé à *_output*.

Lorsqu'on utilise les threads, il faut prendre garde à **ne pas changer le répertoire courant** car cela affecte tous les threads.

---

Merci pour votre attention
==========================

## Des Questions ?

### Ce projet est sous licence Apache 2.0, donc c'est **aussi le vôtre !** Toute contribution est la bienvenue sur <http://github.com/c4s4/neon>

### Slides disponibles à l'adresse <http://sweetohm.net/slides/slides-neon>

### <casa@sweetohm.net>

### <http://sweetohm.net>
