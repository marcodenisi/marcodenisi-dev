---
title: "Perchè dovresti usare Conventional Commits"
date: 2020-01-10T10:30:41+01:00
draft: false
categories: ["Development"]
tags: ["git", "semantic-versioning", "changelog", "automation", "cd"]
---

Ormai ovunque (o quasi) viene utilizzato un sistema di *versioning* per rendere più semplice la collaborazione di diversi sviluppatori sulla stessa *codebase*. Tutto bellissimo, non fosse che nella maggior parte dei casi, a lungo andare la storia dei diversi *commit* si avvicini molto alla seguente:

![Git Commit - xkcd](/img/articles/git_commit.png)

Qualche commit ha introdotto una *breaking change*? Ci sono state *fix*? In bocca al lupo a trovare la risposta guardando solo la storia dei commit: cominciate pure a peregrinare all'interno dell'open space chiedendo chi ha fatto cosa.

Da questo problema ne deriva un altro. Poniamo che si debba decidere il numero di versione del software che si rilascia. Solitamente, la versione è composta da tre numeri `X.Y.Z`. Come si decide che valore dare ad ognuno di questi? Non solo: come si crea il *changelog* al rilascio? 

A queste problematiche c'è una soluzione specifica, ma tutto parte dall'introduzione di **Conventional Commits**.

# Cos'è Conventional Commits

Conventional Commits non è nient'altro che una *convenzione* su come scrivere i messaggi di commit. Come dice il [sito ufficiale](https://www.conventionalcommits.org/en/v1.0.0/), i commit dovrebbero essere nella forma
```
<type>[optional scope]: <description>
[optional body]
[optional footer(s)]
```

Un esempio pratico è il seguente, tralasciando i campi opzionali:
```
feat: add new API to retrieve users
```
La specifica definisce diversi *type*. I più importanti possono essere `feat` che descrive l'introduzione di una nuova feature, oppure `fix`, che invece esplicita l'introduzione di una fix. Ve ne sono molti altri, vi rimando alla documentazione ufficiale se volete entrare nel dettaglio.

# Perchè è utile

Una *git log* a volte vale più di mille parole:
```bash
fix(lambda-article): change publish date
fix(lambda-article): make article not draft anymore
feat(lambda-article): add new article on lambda functions
feat(cd): introduce github action to automatically build website
```
Solo leggendo la serie di *commit descriptions* si riesce a capire esattamente cosa è stato fatto. In questo esempio, sono state introdotte due nuove *feature* ed altrettante *fix*.

Con un po' di consistenza e di buona volontà, si risolve il problema dello storico illeggibile. Tuttavia, niente che non si possa ottenere senza queste convenzioni. 

Ma è introducendo il **Semantic Versioning** che si hanno i benefici maggiori.

# Semantic Versioning

Il [Semantic Versioning](https://semver.org/) è un insieme di regole (un *framework*, come direbbero quelli bravi) che definisce esattamente come decidere il numero di versione di un applicativo. 

Dato un numero di versione `X.Y.Z`:

- X è il MAJOR: si incrementa quando si introducono *breaking change*
- Y è il MINOR: per introduzioni di *feature* retrocompatibili
- Z è il PATCH: quando si inseriscono *bug fix* retrocompatibili

Il legame con Conventional Commits è evidente: dalla storia dei commit si riesce a definire in maniera esatta il numero di versione da assegnare.

Ci sono inoltre dei *tool* che permettono di autogenerare il changelog a partire dalla storia dei commit: in Sky ne abbiamo sviluppato uno internamente (*what-bump*, scritto in Rust), ma le alternative open-source non mancano.

Basta inserire questi tool nella pipeline di rilascio *et voilà*, numero di versione deciso automaticamente e changelog generato.