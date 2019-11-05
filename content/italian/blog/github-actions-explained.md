---
title: "Github Actions: un esempio concreto con Hugo e Latex"
date: 2019-11-04T18:26:59+01:00
draft: true
categories: ["Development"]
tags: ["cd", "github-actions", "github", "hugo"]
---

È un po' di tempo che ho deciso di scrivere il mio curriculum in Latex e di utilizzare Github per il versionamento ([qui](https://github.com/marcodenisi/cv) il repository). Mi sono detto che sarebbe stato molto bello se, ad ogni aggiornamento (aka ad ogni nuovo commit sul repository), mi fossi ritrovato il pdf risultante dalla compilazione come nuova *release* del mio cv.

Lo stesso desiderio l'ho avuto quando ho cominciato a pensare e costruire questo blog: non sarebbe fantastico se ad ogni nuovo contenuto la pubblicazione avvenisse in maniera automatica?

Bene, questo articolo è la storia di questo viaggio: come sono passato dal fare tutto manualmente all'avere un flusso completamente automatizzato con Github Actions.

# Github Actions
Github Actions è una nuova *feature* di Github che permette di creare dei veri e propri workflow di CI/CD all'interno di ogni repository. 

Un **workflow** viene definito da un file *yaml* e non è nient'altro che una serie di **job**. Ogni job, a sua volta, è formato da **step**. Infine, ogni step è formato da un insieme di **action**. Ed è proprio l'action l'unità atomica di Github Actions: ognuna di queste compie una determinata azione (esempio: fa il checkout del repository, crea una release, ...).

All'interno di un singolo repository possono convivere diversi workflows. Questi devono essere definiti all'interno della cartella `.github/workflows`.

Ogni workflow viene *triggerato* da un evento. Ad esempio, potremmo avere un workflow che viene lanciato quando viene fatta una *push* in un *branch*, come mostrato sotto:

```yaml
name: Deploy new version    # nome workflow
on: 
  push:
    branches:
      - master
```
Ma come avviene effettivamente l'esecuzione? Github ha un pool di macchine virtuali (hostate su Azure) sulle quali sono installati dei servizi chiamati *runner*. Quando viene creata l'istanza di workflow, i job al suo interno verranno eseguiti da questi runner, in parallelo o in maniera sequenziale.

Potete trovare [qui](https://help.github.com/en/github/automating-your-workflow-with-github-actions) la documentazione ufficiale.

# CV e Latex
Come far compilare automaticamente un documento in Latex? [Qui](https://github.com/marcodenisi/cv/blob/master/.github/workflows/main.yml) il file che definisce il workflow.

```yaml
name: Build CV
on:
  push:
    tags:
      - 'v*.*.*'
jobs:
  release_cv:
    runs-on: ubuntu-latest
    steps:
    ...
  
  push_to_blog:
    needs: release_cv
    runs-on: ubuntu-latest
    steps:
    ...
```
Questo è un primo estratto per mostrarvi la struttura dello yaml. Qui possiamo trovare la definizione del nome del workflow e dell'evento al quale viene triggerato, nel mio caso il push di un tag. Inoltre, definisco due job:

- `release_cv`: si occupa di compilare i file `.tex` e di creare una nuova release su Github
- `push_to_blog`: data la release, la pusha sul repo di questo blog

Le action che compongono questi job sono abbastanza auto esplicative. La parte interessante secondo me è quella relativa al come i due job interagiscono tra di loro. Ho già spiegato prima di come ogni job viene preso in carico da un runner. Si tratta quindi potenzialmente di servizi diversi su macchine diverse, che non condividono quindi alcun dato. Infatti, per passare informazioni da un job all'altro entrano in gioco gli **artifact**, degli oggetti che vengono salvati e recuperati a runtime.

L'ultima azione del primo job è infatti l'upload di un artifact contenente il pdf:
```
    - name: Upload to virtual env
      uses: actions/upload-artifact@v1
      with:
        name: cv
        path: cv_MarcoDenisi.pdf
```

e la prima azione del secondo job è il download di questo artefatto:
```
    - name: Download from virtual env
      uses: actions/download-artifact@v1
      with:
        name: cv
```

# Blog e Hugo

La compilazione ed upload del blog è invece un po' più casereccia, [qui](https://github.com/marcodenisi/marcodenisi-dev/blob/master/.github/workflows/main.yml) la versione completa:
```yaml
name: Deploy new version
on: 
  push:
    branches:
      - master
      
jobs:
  deploy_blog:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout master
      uses: actions/checkout@v1
    - name: Build blog and deploy
      run:  |
        ... bash script
``` 
Come vedete, ogni azione può anche eseguire dei semplici script bash (che per brevità ometto). Non elegantissimo, ma funzionante!

# Mettiamo tutto insieme

Tutto questo procedimento mi ha portato a:

- dismettere una vecchia (e brutta) soluzione composta da Travis + script bash
- imparare ad usare un nuovo tool
- mantenere aggiornato il CV
- ... e quindi mantenere aggiornato il blog

Attualmente Github Actions è gratuito per tutti i repository, sia pubblici che privati (questi ultimi con qualche riserva). Trovo valga la pena cominciare ad utilizzarlo per avere una soluzione completamente integrata in un unico posto, anche se alcune limitazioni, ad oggi, esistono.