---
title: "Automatiser le test de charge Azure avec GitHub\_Actions"
permalink: index.html
layout: home
---

Les exercices suivants constituent un apprentissage pratique de l’implémentation des actions et des workflows GitHub visant à automatiser l’exécution d’un test de charge avec Test de charge Azure. 

## Exercices
<hr/>


{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions/Labs'" %} {% for activity in labs  %}
* [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) <br/> {{ activity.lab.description }} {% endfor %}
