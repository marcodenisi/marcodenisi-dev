---
title: "Resilienza nell'era dei microservizi"
date: 2020-03-15T12:21:26+01:00
draft: false
categories: ["Development"]
tags: ["resilience4j", "java", "patterns", "circuit-breaker"]
---

Quante volte è capitato di dover integrare un servizio esterno all'interno dei nostri progetti? Sono all'ordine del giorno frasi come:

- la chiamata HTTP ci mette una vita!
- questa roba si spacca una volta si e l'altra pure
- in produzione andrà giù tutto!

Tutte queste (giuste) lamentele sono tipiche di quando dobbiamo fare chiamate tra macchine remote. Soprattutto quando si ha a che fare con sistemi *unresponsive*, c'è il rischio che finiscano le risorse portando a fallimenti a cascata. 

Questo è particolarmente vero nel caso in cui ci troviamo in un'architettura a microservizi con chiamate HTTP sincrone.

# Resilienza

Questa tipologia di problemi esiste ed esisterà sempre. Tuttavia esistono tutta una serie di pattern che ci permettono di mitigarne le conseguenze e quindi di incrementare la resilienza dei nostri servizi.

## Circuit Breaker

L'idea di base è molto semplice: *wrappare* la chiamata remota in un oggetto che monitora i fallimenti. Quando questi raggiungono una certa soglia, il *wrapper* non esegue più la chiamata remota, aprendo il circuito. In questo modo il carico sul server remoto scende, assicurando anche che il nostro servizio sia *responsive*.

## Bulkhead

Un bulkhead (in italiano *paratia*) è una parete che separa diverse sezioni di una nave in modo che incidenti in una sezione (acqua, fuoco, ...) non impattino le sezioni vicine.

Nel mondo dei microservizi, un bulkhead è un pattern che permette di isolare i fallimenti ad un solo componente, in modo da non *tirare giù* l'intera applicazione. Come? Limitando il numero di chiamate concorrenti verso un sistema esterno.

## Rete Limiter

Anche qui l'idea è molto semplice: se si devono fare diverse chiamate verso uno stesso servizio esterno, conviene spalmare queste chiamate all'interno di un lasso temporale in modo da dare il tempo al servizio di servire tutte le richieste.

# Un esempio: resilience4J
Il codice completo dell'esempio è a disposizione a [questo link](https://github.com/marcodenisi/resilience-microservices-era).

[Resilience4j](https://github.com/resilience4j/resilience4j) è una libreria che permette di applicare tutti questi pattern in Java. Prende spunto dalla libreria di Netflix [Hystrix](https://github.com/Netflix/Hystrix), attualmente in *maintenance mode*.

Poniamo di avere un microservizio che espone una API in `GET` dietro l'endpoint `/remote` che va a richiamare un servizio esterno (`ExternalService`):

{{<mermaid align="left">}}
sequenceDiagram

    participant U as User
    participant RC as ResilientController
    participant RS as ResilientService
    participant ES as ExternalService
    U ->> RC: GET /remote
    RC ->> RS: remoteCall()
    RS -->> ES: remoteCall()
    ES ->> ES: compute
    ES -->> RS: RemoteCallResult
    RS ->> RC: RemoteCallResult
    RC ->> U: RemoteCallResult
{{</mermaid>}}

Resilience4J ci permette di decorare questa chiamata per aggiungere tutte le funzionalità aggiuntive del caso. Ad esempio, se volessimo aggiungere un `CircuitBreaker`:

```java
@Service
public class ResilientService {

    @Autowired private ExternalService externalService;
    @Autowired private CircuitBreaker circuitBreaker;

    public RemoteCallResult getRemote() {
        return Decorators.ofSupplier(externalService::remoteCall)   // wraps the external call
                .withCircuitBreaker(circuitBreaker)                 // decorate with a circuit breaker
                .get();                                             // perform the call
    }
}
```

Quando andremo a chiamare questo metodo, possono succedere diverse cose, in base allo stato del circuit breaker:

- stato `CLOSED`: la chiamata al servizio esterno viene fatta
- stato `OPEN`: la chiamata non viene fatta, viene lanciata una `CallNotPermittedException`
- stato `HALF_OPEN`: è passato del tempo dall'ultimo fallimento, solo un numero limitato di chiamate viene effettuato

Il circuit breaker può essere configurato a piacimento, vi rimando alla [documentazione ufficiale](https://resilience4j.readme.io/docs/circuitbreaker) per tutti i dettagli del caso. Nel mio esempio, ho scritto la configurazione in un file *yaml*:

```yaml
resilience4j.circuitbreaker:
    instances:
        remoteService:                                  // circuit breaker instance name
            registerHealthIndicator: true
            slidingWindowType: TIME_BASED               // time based check
            slidingWindowSize: 10                       // last 10 seconds
            permittedNumberOfCallsInHalfOpenState: 3
            minimumNumberOfCalls: 10                    // minimum number of calls to create a statistic
            waitDurationInOpenState: 30s
            failureRateThreshold: 50                
```

# Conclusione

Introdurre pattern come il circuit breaker, il bulkhead ed altri permette ai sistemi di essere più resilienti ai fallimenti di servizi esterni ed interni, mettendo anche in campo strategie per tornare ad uno stato di normalità.

Resilience4J è una libreria molto facile da usare ed integrare per implementare questi pattern. Inoltre, è facile aggiungere l'integrazione con Grafana per un miglior *monitoring*.

<script async src="https://unpkg.com/mermaid@8.2.3/dist/mermaid.min.js"></script>