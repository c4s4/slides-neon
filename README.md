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
