---
title: "Lambda Step by Step"
date: 2019-09-26T19:07:46+02:00
draft: false
categories: ["Development"]
tags: ["aws", "lambda", "sam", "java8"]
---

Nelle ultime settimane ho avuto molto a che fare con la creazione di [AWS Lambda](https://aws.amazon.com/it/lambda/). 

La difficoltà più grossa con cui ho dovuto fare i conti è stata quella di testare in locale queste funzioni. Sicuramente è possibile (anzi, necessario) scrivere unit ed integration test, ma mi piace comunque testare in un ambiente il più vicino possibile a quello di produzione.

Vorrei quindi costruire una sorta di guida per avere un reminder per i futuri sviluppi. Per fare ciò sono necessari i seguenti step:

- scrivere una lambda che *fa cose*
- con l'aiuto di SAM (e la sua CLI), invocare e debuggare la funzione

Let's start!

# SAM
SAM (*Serverless Application Model*) è un framework open source per definire, testare, ed infine *deployare* applicazioni *serverless*. Il tutto scrivendo un file *.yaml* (template) che SAM si occuperà di trasformare in sintassi AWS CloudFormation.

In questo caso, ci servirà per definire l'endpoint al quale la lambda risponderà. Tramite la sua CLI, riusciremo anche a lanciare (e debuggare) la lambda stessa in locale.

Un po' di documentazione nella [pagina ufficiale](https://aws.amazon.com/it/serverless/sam/).

# Creazione delle Lambda

La SAM CLI offre una funzionalità molto comoda per creare uno stub di lambda. Basterà infatti lanciare 
```bash
sam init --runtime java8 --name StuffApi
```
per creare un intero progetto maven con delle classi di esempio. In particolare, verrà creata questa struttura:
```bash
StuffApi
├── README.md 
├── pom.xml
├── src
│   ├── main
│   │   └── java
│   │       └── helloworld
│   │           ├── App.java              <-- Lambda
│   │           └── GatewayResponse.java  <-- POJO per risposta API Gateway 
│   └── test
│       └── java
│           └── helloworld
│               └── AppTest.java
└── template.yaml
```
Vi consiglio di dare una lettura soprattutto al file `README.md`, che contiene molte indicazione utili sui prossimi passi da seguire.

Implementiamo una semplice lambda che restituisce un oggetto `User`:
```java 
public class StuffHandler implements RequestHandler<Void, GatewayResponse<UserDto>> {
    @Override
    public GatewayResponse<UserDto> handleRequest(final Void input, final Context context) {
        Map<String, String> headers = new HashMap<>();
        headers.put("Content-Type", "application/json");
        return new GatewayResponse<>(new UserDto.UserDtoBuilder().setEmail("foo@bar.com").setUsername("foobar").build(),
                headers,
                200);
    }
}
```
La risposta è *wrappata* all'interno dell'oggetto `GatewayResponse`. Questo oggetto, contiene tre campi:

- `body`: la risposta vera e propria della lambda
- `headers`: eventuali header che vogliamo dare in risposta (in questo caso ho aggiunto l'header `Content-Type`)
- `statusCode`: il codice HTTP restituito (nel mio caso, 200)

# Creazione del SAM template
Veniamo ora alla "ciccia", il SAM template. Ecco la prima parte del template yaml:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
    StuffApi
    Sample SAM Template for StuffApi
Globals:
    Function:
        Timeout: 20
```
Qui sono contenute delle informazioni generiche che servono a SAM per parsare il file (`AWSTemplateFormatVersion` ad esempio). La parte più interessante riguarda la sezione `Globals`: qui infatti possiamo definire tutte le variabili globali della nostra lambda.
```yaml
Resources:
  StuffFunction:
        Type: AWS::Serverless::Function
        Properties:
            CodeUri: target/StuffApi-1.0.jar
            Handler: dev.marcodenisi.stuff.StuffHandler::handleRequest
            Runtime: java8
            Events:
                Stuff:
                    Type: Api
                    Properties:
                        Path: /stuff
                        Method: get
```
In questa seconda sezione, definiamo la *Lambda Function* vera e propria. Le diamo un nome (`StuffFunction`), definiamo il nome del *jar* contenente il codice sorgente ed anche la classe `handler`, con relativo metodo che si occuperà di servire il contenuto.

Sotto `Events`, definiamo un API Gateway (`Type: Api`) che risponde a chiamate HTTP GET (`Method: get`) al *path* `/stuff`.

# Invocare e debuggare la funzione

Per prima cosa, *pacchettizziamo* la lambda. Nel mio caso, usando Java + Maven, basta lanciare `mvn package` per creare il *jar*.

Per invocare la lambda, utilizziamo il comando di `invoke` della SAM cli.
```bash
sam local invoke "StuffFunction" --no-event
... // log "di servizio"
{"body":{"username":"foobar","email":"foo@bar.com"},"headers":{"Content-Type":"application/json"},"statusCode":200}
```
Questo comando non fa nient'altro che creare un piccolo container docker. Infatti, il primo start sarà piuttosto lento a causa del download dell'immagine, per poi velocizzarsi le volte successive. [Ecco](https://hub.docker.com/r/lambci/lambda/) la documentazione dell'immagine docker utilizzata.

L'opzione `--no-event` specifica che alla lambda non serve nulla in ingresso. Nel caso dovessero servire degli eventi specifici, il comando `sam local generate-event` è utile per creare degli eventi di esempio.

Per debuggare, basta aggiungere un flag:
```bash
sam local invoke "StuffFunction" --no-event -d 5858
```
In questo modo, la lambda non verrà eseguita fino a che un processo si attaccherà in debug alla porta specificata (la 5858, nel mio caso).

![That's all folks!](/gifs/thats_all_folks.gif)