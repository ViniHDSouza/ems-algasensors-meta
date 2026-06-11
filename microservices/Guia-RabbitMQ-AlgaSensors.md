# Guia Completo de RabbitMQ — Aprendendo com o Projeto AlgaSensors

> **Autor:** Seu Mentor com +10 anos de experiência em mensageria e sistemas distribuídos
>
> **Nível:** Iniciante absoluto — sem pré-requisitos
>
> **Projeto de referência:** AlgaSensors (plataforma de monitoramento de sensores de temperatura)

---

## Sumário

1. [O Que é RabbitMQ e Por Que Ele Existe?](#1-o-que-é-rabbitmq-e-por-que-ele-existe)
2. [Os Conceitos Fundamentais (Vocabulário do RabbitMQ)](#2-os-conceitos-fundamentais-vocabulário-do-rabbitmq)
3. [A Arquitetura do AlgaSensors — O Panorama Geral](#3-a-arquitetura-do-algasensors--o-panorama-geral)
4. [Subindo o RabbitMQ com Docker](#4-subindo-o-rabbitmq-com-docker)
5. [Conectando a Aplicação ao RabbitMQ](#5-conectando-a-aplicação-ao-rabbitmq)
6. [O Produtor — Quem Envia as Mensagens](#6-o-produtor--quem-envia-as-mensagens)
7. [A Exchange Fanout — O "Alto-Falante"](#7-a-exchange-fanout--o-alto-falante)
8. [As Filas (Queues) — As "Caixas de Correio"](#8-as-filas-queues--as-caixas-de-correio)
9. [Os Bindings — As "Conexões" entre Exchange e Filas](#9-os-bindings--as-conexões-entre-exchange-e-filas)
10. [O Consumidor — Quem Recebe as Mensagens](#10-o-consumidor--quem-recebe-as-mensagens)
11. [Concorrência e Prefetch — Controlando a Velocidade](#11-concorrência-e-prefetch--controlando-a-velocidade)
12. [Serialização — Transformando Objetos em Mensagens](#12-serialização--transformando-objetos-em-mensagens)
13. [O Inicializador — Garantindo que Tudo Existe](#13-o-inicializador--garantindo-que-tudo-existe)
14. [Fluxo Completo Passo a Passo](#14-fluxo-completo-passo-a-passo)
15. [Tipos de Exchange — Além do Fanout](#15-tipos-de-exchange--além-do-fanout)
16. [Glossário Visual](#16-glossário-visual)
17. [Exercícios Práticos](#17-exercícios-práticos)
18. [Erros Comuns de Iniciantes](#18-erros-comuns-de-iniciantes)
19. [Próximos Passos](#19-próximos-passos)

---

## 1. O Que é RabbitMQ e Por Que Ele Existe?

### A analogia dos Correios

Imagine que você tem uma empresa que precisa enviar cartas para vários departamentos. Você poderia ir pessoalmente a cada departamento entregar a carta (comunicação direta), mas isso tem problemas:

- **E se o departamento estiver fechado?** Sua carta se perde.
- **E se você tiver 1000 cartas para entregar?** Você vai ficar parado esperando cada departamento receber.
- **E se um novo departamento precisar receber a mesma carta?** Você precisa mudar seu processo inteiro.

Agora imagine que existe uma **agência dos Correios** no meio. Você entrega a carta lá, e os Correios se encarregam de distribuir para todos os departamentos certos, na hora certa, sem perder nada.

**O RabbitMQ é essa agência dos Correios para os seus programas.**

### No mundo real dos programas

Sem RabbitMQ, seu sistema se parece com isto:

```
[Serviço A] ---chamada direta---> [Serviço B]
                                   ↑
                      Se B está fora, A quebra também!
```

Com RabbitMQ:

```
[Serviço A] --envia mensagem--> [RabbitMQ] --entrega quando possível--> [Serviço B]
                                                                        [Serviço C]
                                                                        [Serviço D]
```

### Benefícios concretos

- **Desacoplamento:** quem envia não precisa saber quem recebe.
- **Resiliência:** se o receptor estiver fora do ar, a mensagem fica guardada na fila.
- **Escalabilidade:** você pode ter vários consumidores processando a mesma fila em paralelo.
- **Flexibilidade:** adicionar um novo consumidor não exige mudar quem envia.

---

## 2. Os Conceitos Fundamentais (Vocabulário do RabbitMQ)

Antes de mergulhar no código, vamos aprender as "palavras" que o RabbitMQ usa. Pense nisso como aprender o vocabulário de um idioma novo antes de montar frases.

| Conceito | O que é | Analogia |
|---|---|---|
| **Producer** (Produtor) | O programa que ENVIA a mensagem | A pessoa que escreve e posta a carta |
| **Exchange** | O roteador que DISTRIBUI a mensagem | A agência dos Correios que decide para onde a carta vai |
| **Queue** (Fila) | A caixa onde a mensagem ESPERA | A caixa de correio do destinatário |
| **Binding** | A regra que CONECTA uma exchange a uma fila | O endereço escrito no envelope |
| **Consumer** (Consumidor) | O programa que RECEBE a mensagem | A pessoa que abre a caixa de correio e lê a carta |
| **Message** (Mensagem) | O dado sendo transportado | A carta em si |
| **Routing Key** | Uma etiqueta na mensagem usada para roteamento | O CEP da carta |

### O caminho de uma mensagem

```
Producer → Exchange → Binding → Queue → Consumer
```

Toda mensagem passa por esse caminho. Não existe atalho. Mesmo quando parece que você está enviando "direto para a fila", na verdade existe uma exchange padrão invisível no meio.

---

## 3. A Arquitetura do AlgaSensors — O Panorama Geral

O projeto AlgaSensors é composto por **3 microsserviços**:

| Microsserviço | Porta | Função | Usa RabbitMQ? |
|---|---|---|---|
| `device-management` | 8080 | Cadastra e gerencia os sensores | Não |
| `temperature-processing` | 8081 | Recebe leituras de temperatura e publica no RabbitMQ | Sim (Produtor) |
| `temperature-monitoring` | 8082 | Consome mensagens, monitora e gera alertas | Sim (Consumidor) |

### O fluxo simplificado

```
Sensor IoT envia temperatura via HTTP POST
            │
            ▼
┌──────────────────────────┐
│  temperature-processing  │  ← recebe a leitura
│       (porta 8081)       │
│                          │
│  Publica no RabbitMQ     │
└───────────┬──────────────┘
            │
            ▼
┌──────────────────────────┐
│       RABBITMQ           │
│                          │
│  Exchange (Fanout)       │
│  "temperature-processing │
│   .temperature-received  │
│   .v1.e"                 │
│       │          │       │
│       ▼          ▼       │
│   [Fila 1]   [Fila 2]   │
│  process-    alerting    │
│  temperature             │
└───┬──────────────┬───────┘
    │              │
    ▼              ▼
┌──────────────────────────┐
│ temperature-monitoring   │  ← consome as mensagens
│      (porta 8082)        │
│                          │
│  Listener 1: atualiza    │
│  a temperatura do sensor │
│                          │
│  Listener 2: verifica    │
│  se precisa gerar alerta │
└──────────────────────────┘
```

### Por que essa arquitetura é inteligente?

Repare que a **mesma mensagem** (uma leitura de temperatura) precisa ser processada de **duas formas diferentes**:
1. Atualizar o registro do sensor no banco de dados.
2. Verificar se a temperatura ultrapassou um limite para gerar alerta.

Sem RabbitMQ, o `temperature-processing` precisaria chamar dois endpoints diferentes do `temperature-monitoring`. Com RabbitMQ e uma **exchange Fanout**, ele publica UMA vez e a mensagem é automaticamente copiada para as duas filas.

---

## 4. Subindo o RabbitMQ com Docker

O projeto usa Docker para subir o RabbitMQ. Vamos analisar o arquivo `docker-compose.yml`:

```yaml
services:
  algasensors-rabbitmq:
    image: rabbitmq:3-management     # ①
    restart: no
    ports:
      - 5672:5672                    # ②
      - 15672:15672                  # ③
    environment:
      RABBITMQ_DEFAULT_USER: rabbitmq    # ④
      RABBITMQ_DEFAULT_PASS: rabbitmq    # ④
    volumes:
      - algasensors-rabbitmq:/var/lib/rabbitmq/   # ⑤

volumes:
  algasensors-rabbitmq:
```

### Explicação linha por linha

| # | O que faz | Detalhe |
|---|---|---|
| ① | Usa a imagem oficial com painel de administração | A tag `3-management` inclui uma interface web para você visualizar filas, exchanges e mensagens |
| ② | Porta 5672 — protocolo AMQP | É por aqui que suas aplicações Java se comunicam com o RabbitMQ |
| ③ | Porta 15672 — painel web | Acesse `http://localhost:15672` no navegador para ver tudo funcionando |
| ④ | Usuário e senha | Ambos definidos como `rabbitmq` (em produção, use senhas fortes!) |
| ⑤ | Volume para persistência | Garante que as filas e mensagens não se perdem quando o container reinicia |

### Como rodar

```bash
# Na pasta raiz do projeto (onde está o docker-compose.yml)
docker-compose up -d

# Verificar se subiu
docker-compose ps

# Acessar o painel web
# Abra no navegador: http://localhost:15672
# Login: rabbitmq / rabbitmq
```

### O que você verá no painel web

O painel de gerenciamento é o seu **melhor amigo** enquanto aprende. Nele você pode:
- Ver quantas mensagens estão em cada fila
- Ver quais exchanges existem
- Ver os bindings (conexões entre exchanges e filas)
- Publicar mensagens manualmente para testar
- Ver a taxa de mensagens por segundo

---

## 5. Conectando a Aplicação ao RabbitMQ

### A dependência no Gradle

Nos arquivos `build.gradle` de ambos os microsserviços que usam RabbitMQ, existe esta linha:

```groovy
implementation 'org.springframework.boot:spring-boot-starter-amqp'
```

Essa é a **única dependência** que você precisa. O Spring Boot AMQP traz tudo: a biblioteca do RabbitMQ, as abstrações do Spring e a configuração automática.

> **AMQP** = Advanced Message Queuing Protocol. É o "idioma" que o RabbitMQ fala. Assim como HTTP é o idioma da web, AMQP é o idioma das filas de mensagens.

### A configuração de conexão

No `application.yml` do `temperature-processing` (produtor):

```yaml
spring:
  rabbitmq:
    host: localhost        # onde o RabbitMQ está rodando
    port: 5672             # porta AMQP (NÃO é a 15672 do painel)
    username: rabbitmq     # mesmo usuário do docker-compose
    password: rabbitmq     # mesma senha do docker-compose
```

No `application.yml` do `temperature-monitoring` (consumidor), além da conexão, temos uma configuração extra:

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: rabbitmq
    password: rabbitmq
    listener:
      simple:
        prefetch: 4        # ← veremos isso em detalhes mais à frente
```

O Spring Boot faz a **conexão automaticamente** quando a aplicação inicia. Você não precisa escrever código para conectar — basta configurar no YAML.

---

## 6. O Produtor — Quem Envia as Mensagens

O produtor no projeto AlgaSensors é o `TemperatureProcessingController`. Vamos analisá-lo parte por parte.

### O código completo

```java
@RestController
@RequestMapping("/api/sensors/{sensorId}/temperatures/data")
@RequiredArgsConstructor
public class TemperatureProcessingController {

    private final RabbitTemplate rabbitTemplate;   // ①

    @PostMapping(consumes = MediaType.TEXT_PLAIN_VALUE)
    public void data(@PathVariable TSID sensorId, @RequestBody String input) {

        // Validações omitidas para foco...

        Double temperature = Double.parseDouble(input);

        // ② Monta o objeto da mensagem
        TemperatureLogOutput logOutput = TemperatureLogOutput.builder()
                .id(IdGenerator.generateTimeBasedUUID())
                .sensorId(sensorId)
                .value(temperature)
                .registeredAt(OffsetDateTime.now())
                .build();

        // ③ Define os parâmetros de envio
        String exchange = FANOUT_EXCHANGE_NAME;
        String routingKey = "";
        Object payload = logOutput;

        // ④ Adiciona informação extra no cabeçalho
        MessagePostProcessor messagePostProcessor = message -> {
            message.getMessageProperties()
                   .setHeader("sensorId", logOutput.getSensorId().toString());
            return message;
        };

        // ⑤ ENVIA a mensagem!
        rabbitTemplate.convertAndSend(exchange, routingKey, payload, messagePostProcessor);
    }
}
```

### Explicação detalhada

**① RabbitTemplate** — É o "carteiro" do Spring. Ele é a classe principal para enviar mensagens ao RabbitMQ. O Spring cria e configura esse objeto automaticamente para você (graças à configuração no `application.yml`).

**② O objeto da mensagem** — A mensagem que será enviada é um objeto Java normal (`TemperatureLogOutput`). Ele contém: o ID único da leitura, o ID do sensor, o valor da temperatura e o horário do registro.

```java
public class TemperatureLogOutput {
    private UUID id;
    private TSID sensorId;
    private OffsetDateTime registeredAt;
    private Double value;
}
```

**③ Os parâmetros de envio:**
- `exchange`: o nome da exchange para onde enviar (no caso, `"temperature-processing.temperature-received.v1.e"`)
- `routingKey`: uma chave de roteamento. Para exchange Fanout, é `""` (vazio), pois o Fanout ignora a routing key
- `payload`: o objeto que será convertido em JSON e enviado

**④ MessagePostProcessor** — Permite adicionar informações extras no "envelope" da mensagem (os headers). Aqui, ele adiciona o `sensorId` como header. Isso é útil para que o consumidor possa filtrar ou logar sem precisar abrir o corpo da mensagem.

**⑤ O envio** — `convertAndSend` faz três coisas em uma chamada:
1. **Convert**: converte o objeto Java para JSON (usando Jackson)
2. **And**: aplica o `MessagePostProcessor` (adiciona os headers)
3. **Send**: envia para o RabbitMQ

### Analogia

Imagine que você está em uma loja de presentes:
- O **objeto** é o presente
- O **Jackson** embala o presente em uma caixa (JSON)
- O **MessagePostProcessor** cola uma etiqueta na caixa
- O **RabbitTemplate** entrega a caixa nos Correios (exchange)
- A **routing key** é o CEP (que no Fanout é ignorado — todos recebem)

---

## 7. A Exchange Fanout — O "Alto-Falante"

### O que é uma Exchange?

A Exchange é o componente que **decide para onde a mensagem vai**. O produtor NUNCA envia direto para uma fila. Ele sempre envia para uma Exchange, e ela distribui.

### O tipo Fanout

No AlgaSensors, a exchange é do tipo **Fanout**. "Fan out" significa "espalhar para todos". É como um alto-falante: quando você fala nele, todo mundo que está ouvindo recebe a mesma mensagem.

### O código da Exchange (lado do Produtor)

No `RabbitMQConfig.java` do `temperature-processing`:

```java
@Configuration
public class RabbitMQConfig {

    public static final String FANOUT_EXCHANGE_NAME =
        "temperature-processing.temperature-received.v1.e";   // ①

    @Bean
    public FanoutExchange exchange() {                         // ②
        return ExchangeBuilder
            .fanoutExchange(FANOUT_EXCHANGE_NAME)
            .build();
    }
}
```

**① A convenção de nomes:** O nome da exchange segue um padrão profissional:
```
temperature-processing   → microsserviço que publica
.temperature-received    → o evento que aconteceu
.v1                      → versão (para evolução futura)
.e                       → indica que é uma exchange
```

**② `@Bean`:** No Spring, `@Bean` significa "crie este objeto e guarde para uso futuro". Quando a aplicação inicia, o Spring cria a `FanoutExchange` e a registra no RabbitMQ.

### Por que Fanout e não outro tipo?

Porque neste caso, TODOS os consumidores precisam receber TODAS as mensagens. Cada leitura de temperatura precisa ser processada tanto para atualização quanto para alertas. O Fanout é perfeito para esse cenário de "broadcast" (transmissão para todos).

---

## 8. As Filas (Queues) — As "Caixas de Correio"

### O código das Filas (lado do Consumidor)

No `RabbitMQConfig.java` do `temperature-monitoring`:

```java
@Configuration
public class RabbitMQConfig {

    public static final String QUEUE_PROCESS_TEMPERATURE =
        "temperature-monitoring.process-temperature.v1.q";     // ①

    public static final String QUEUE_ALERTING =
        "temperature-monitoring.alerting.v1.q";                // ②

    @Bean
    public Queue queueProcessTemperature() {
        return QueueBuilder.durable(QUEUE_PROCESS_TEMPERATURE).build();   // ③
    }

    @Bean
    public Queue queueAlerting() {
        return QueueBuilder.durable(QUEUE_ALERTING).build();
    }
}
```

**① e ② Nomes das filas:** Seguem a mesma convenção:
```
temperature-monitoring       → microsserviço consumidor
.process-temperature         → o que essa fila faz
.v1                          → versão
.q                           → indica que é uma queue (fila)
```

**③ `durable`:** Significa que a fila **sobrevive a reinicializações** do RabbitMQ. Se o servidor cair e voltar, a fila ainda estará lá com suas mensagens. Isso é crucial em produção!

### Fila durável vs. não-durável

| Tipo | O que acontece se o RabbitMQ reiniciar? | Quando usar? |
|---|---|---|
| **Durable** (durável) | A fila e suas mensagens são preservadas | Produção (dados que não podem ser perdidos) |
| **Non-durable** | A fila e suas mensagens são perdidas | Filas temporárias, testes |

### Por que duas filas separadas?

Cada fila representa uma **responsabilidade diferente**:
- `process-temperature.q` → atualizar o banco de dados com a temperatura mais recente
- `alerting.q` → verificar se a temperatura ultrapassou limites e gerar alertas

Se usássemos apenas UMA fila, um único consumidor receberia a mensagem e o outro ficaria sem. Com duas filas, cada uma recebe sua própria cópia da mensagem.

---

## 9. Os Bindings — As "Conexões" entre Exchange e Filas

O Binding é a **cola** que conecta uma Exchange a uma Fila. Sem binding, a mensagem chega na exchange e... desaparece, porque ela não sabe para onde enviar.

### O código dos Bindings

```java
// Ainda dentro do RabbitMQConfig.java do temperature-monitoring

public FanoutExchange exchange() {
    return ExchangeBuilder
        .fanoutExchange("temperature-processing.temperature-received.v1.e")
        .build();
}

@Bean
public Binding bindingProcessTemperature() {
    return BindingBuilder
        .bind(queueProcessTemperature())   // ← "Pegue esta fila..."
        .to(exchange());                   // ← "...e conecte a esta exchange"
}

@Bean
public Binding bindingAlerting() {
    return BindingBuilder
        .bind(queueAlerting())
        .to(exchange());
}
```

### Visualizando os Bindings

```
                    ┌───────────────────────┐
                    │    Fanout Exchange     │
                    │  temperature-processing│
                    │  .temperature-received │
                    │  .v1.e                 │
                    └───────┬───────┬────────┘
           Binding ①│               │Binding ②
                    ▼               ▼
   ┌─────────────────────┐  ┌─────────────────────┐
   │ temperature-monitoring│  │ temperature-monitoring│
   │ .process-temperature  │  │ .alerting             │
   │ .v1.q                 │  │ .v1.q                 │
   └───────────────────────┘  └───────────────────────┘
```

### Observação importante

Repare que a exchange é **declarada nos dois microsserviços**: no produtor (onde ela é criada) e no consumidor (onde ela é referenciada para criar os bindings). O RabbitMQ é inteligente — se a exchange já existe, ele simplesmente reutiliza. Não cria duplicata.

---

## 10. O Consumidor — Quem Recebe as Mensagens

### O código do Listener

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class RabbitMQListener {

    private final TemperatureMonitoringService temperatureMonitoringService;
    private final SensorAlertService sensorAlertService;

    @RabbitListener(queues = QUEUE_PROCESS_TEMPERATURE, concurrency = "2-3")   // ①
    @SneakyThrows
    public void handleProcessingTemperature(@Payload TemperatureLogData temperatureLogData) {
        temperatureMonitoringService.processTemperatureReading(temperatureLogData);   // ②
        Thread.sleep(Duration.ofSeconds(5));   // ③
    }

    @RabbitListener(queues = QUEUE_ALERTING, concurrency = "2-3")   // ④
    @SneakyThrows
    public void handleAlerting(@Payload TemperatureLogData temperatureLogData) {
        sensorAlertService.handleAlert(temperatureLogData);   // ⑤
        Thread.sleep(Duration.ofSeconds(5));
    }
}
```

### Explicação detalhada

**① `@RabbitListener`** — Esta é a anotação mágica. Ela diz ao Spring: "Fique escutando esta fila. Quando chegar uma mensagem, chame este método automaticamente." Você não precisa escrever nenhum loop, nenhum polling. O Spring cuida de tudo.

**② Lógica de negócio** — O método `processTemperatureReading` atualiza o monitoramento do sensor:
- Busca o sensor no banco pelo ID
- Se o sensor existe e está habilitado, atualiza a última temperatura e salva o log
- Se não existe ou está desabilitado, ignora a mensagem

**③ `Thread.sleep(5 segundos)`** — Este é um delay artificial para fins de estudo. Em produção, você não teria isso. Serve para simular um processamento demorado e visualizar as mensagens se acumulando na fila.

**④ Segundo listener** — O mesmo princípio, mas escutando a fila de alertas. Note que os dois listeners estão na **mesma classe**, mas escutam filas **diferentes**.

**⑤ Lógica de alerta** — O método `handleAlert` verifica:
- Se existe uma configuração de alerta para aquele sensor
- Se a temperatura está acima do máximo configurado → gera alerta
- Se a temperatura está abaixo do mínimo configurado → gera alerta
- Caso contrário → ignora

### `@Payload` — Deserialização automática

A anotação `@Payload` diz ao Spring: "Pegue o corpo da mensagem (que é JSON) e converta de volta para o objeto Java `TemperatureLogData`." Isso funciona automaticamente graças ao `Jackson2JsonMessageConverter` configurado no `RabbitMQConfig`.

---

## 11. Concorrência e Prefetch — Controlando a Velocidade

### O que é Concorrência?

No `@RabbitListener`, o parâmetro `concurrency = "2-3"` define:

```
concurrency = "2-3"
                │ │
                │ └─ Máximo de 3 threads (consumidores) simultâneos
                └─── Mínimo de 2 threads (consumidores) simultâneos
```

Isso significa que o Spring vai criar entre 2 e 3 **consumidores paralelos** para cada fila. Se a fila tiver muitas mensagens acumuladas, ele sobe para 3. Se estiver tranquilo, usa 2.

### Analogia

Imagine um restaurante: a fila de pedidos é a Queue. Os cozinheiros são os consumidores. Com `concurrency = "2-3"`, você tem entre 2 e 3 cozinheiros trabalhando simultaneamente. Se o restaurante lotar, o gerente coloca o terceiro cozinheiro para ajudar.

### O que é Prefetch?

No `application.yml` do consumidor:

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 4
```

O `prefetch` controla **quantas mensagens cada consumidor "reserva" de uma vez**. Com `prefetch: 4`, cada consumidor pega até 4 mensagens da fila antes de processá-las.

### Por que o Prefetch importa?

```
Sem prefetch (prefetch=ilimitado):
  Consumidor A pega TODAS as 1000 mensagens → sobrecarregado
  Consumidor B fica ocioso → desperdiçado

Com prefetch=4:
  Consumidor A pega 4 mensagens → processa
  Consumidor B pega 4 mensagens → processa
  A terminou? Pega mais 4.
  → Distribuição equilibrada!
```

### Conta rápida no AlgaSensors

Com `concurrency = "2-3"` e `prefetch = 4`:
- No mínimo: 2 consumidores × 4 mensagens = **8 mensagens** sendo processadas/reservadas por fila
- No máximo: 3 consumidores × 4 mensagens = **12 mensagens** sendo processadas/reservadas por fila

---

## 12. Serialização — Transformando Objetos em Mensagens

O RabbitMQ não entende objetos Java. Ele só entende **bytes**. Precisamos converter o objeto Java para algo transportável (como JSON) e depois converter de volta no destino. Esse processo se chama **serialização** e **deserialização**.

### O conversor Jackson

Nos dois microsserviços, o `RabbitMQConfig` configura:

```java
@Bean
public Jackson2JsonMessageConverter jackson2JsonMessageConverter(ObjectMapper objectMapper) {
    return new Jackson2JsonMessageConverter(objectMapper);
}
```

Isso diz ao Spring: "Quando enviar mensagens, converta os objetos para JSON. Quando receber, converta de JSON de volta para objetos."

### O que acontece por trás das cortinas

**Na hora de ENVIAR (produtor):**

```
Objeto Java                          Mensagem no RabbitMQ
─────────────                        ────────────────────
TemperatureLogOutput {         →     {
  id: "abc-123",                       "id": "abc-123",
  sensorId: "0HF3YJKPXS9ED",          "sensorId": "0HF3YJKPXS9ED",
  value: 25.5,                         "value": 25.5,
  registeredAt: "2025-03-19..."        "registeredAt": "2025-03-19..."
}                                    }
```

**Na hora de RECEBER (consumidor):**

```
Mensagem JSON no RabbitMQ      →     Objeto Java
{                                    TemperatureLogData {
  "id": "abc-123",                     id: "abc-123",
  "sensorId": "0HF3YJKPXS9ED",        sensorId: "0HF3YJKPXS9ED",
  "value": 25.5,                       value: 25.5,
  "registeredAt": "2025-03-19..."      registeredAt: "2025-03-19..."
}                                    }
```

> **Nota:** Repare que o produtor usa `TemperatureLogOutput` e o consumidor usa `TemperatureLogData`. São classes **diferentes** em projetos **diferentes**, mas com os **mesmos campos**. Isso é proposital — cada microsserviço define seus próprios modelos, mantendo a independência.

---

## 13. O Inicializador — Garantindo que Tudo Existe

### O problema

Quando sua aplicação inicia, ela precisa que as filas e exchanges já existam no RabbitMQ. Mas e se for a primeira vez rodando? E se alguém deletou a fila manualmente?

### A solução: RabbitMQInitializer

```java
@Configuration
@RequiredArgsConstructor
public class RabbitMQInitializer {

    private final RabbitAdmin rabbitAdmin;   // ①

    @PostConstruct                          // ②
    public void init() {
        rabbitAdmin.initialize();            // ③
    }
}
```

**① RabbitAdmin** — É a classe administrativa do Spring que sabe como criar filas, exchanges e bindings no RabbitMQ.

**② `@PostConstruct`** — Significa "execute este método logo após o objeto ser criado." É uma das primeiras coisas que rodam quando a aplicação inicia.

**③ `initialize()`** — Percorre todos os `@Bean` do tipo Queue, Exchange e Binding que você declarou no `RabbitMQConfig` e os cria no RabbitMQ, caso ainda não existam.

### O efeito prático

Toda vez que a aplicação inicia:
1. O Spring encontra os beans de Queue, Exchange e Binding
2. O `RabbitAdmin.initialize()` se conecta ao RabbitMQ
3. Cria tudo que não existir (e não faz nada com o que já existe)
4. Sua aplicação está pronta para enviar/receber mensagens

---

## 14. Fluxo Completo Passo a Passo

Vamos acompanhar uma leitura de temperatura desde o sensor até o alerta:

### Passo 1: Sensor envia a leitura

```bash
# Um sensor IoT faz um POST para o temperature-processing
curl -X POST http://localhost:8081/api/sensors/0HF3YJKPXS9ED/temperatures/data \
  -H "Content-Type: text/plain" \
  -d "35.7"
```

### Passo 2: Controller recebe e publica

O `TemperatureProcessingController`:
1. Recebe o valor `"35.7"` como texto
2. Converte para `Double` (35.7)
3. Cria o objeto `TemperatureLogOutput` com ID, sensorId, valor e timestamp
4. Usa o `RabbitTemplate` para enviar para a exchange `temperature-processing.temperature-received.v1.e`

### Passo 3: Exchange distribui

A exchange Fanout recebe a mensagem e copia para **todas** as filas conectadas:
- Cópia 1 → fila `temperature-monitoring.process-temperature.v1.q`
- Cópia 2 → fila `temperature-monitoring.alerting.v1.q`

### Passo 4: Consumidores processam

**Listener 1** (`handleProcessingTemperature`):
1. Pega a mensagem da fila `process-temperature`
2. O Jackson converte o JSON de volta para `TemperatureLogData`
3. Chama `temperatureMonitoringService.processTemperatureReading()`
4. O serviço busca o sensor no banco, atualiza a última temperatura, salva o log

**Listener 2** (`handleAlerting`):
1. Pega a mensagem da fila `alerting`
2. O Jackson converte o JSON de volta para `TemperatureLogData`
3. Chama `sensorAlertService.handleAlert()`
4. O serviço verifica se 35.7 está acima do máximo ou abaixo do mínimo configurado
5. Se estiver, gera um log de alerta

### Passo 5: Mensagens são removidas das filas

Após o processamento bem-sucedido, o Spring envia um **acknowledgment** (confirmação) ao RabbitMQ dizendo "recebi e processei com sucesso". O RabbitMQ então remove a mensagem da fila.

> **Se o processamento falhar** (exceção), o RabbitMQ recoloca a mensagem na fila para ser tentada novamente (comportamento padrão).

---

## 15. Tipos de Exchange — Além do Fanout

O AlgaSensors usa apenas Fanout, mas é importante conhecer os outros tipos:

### Fanout (usado no AlgaSensors)

```
Mensagem → [Fanout Exchange] → Fila A (recebe)
                              → Fila B (recebe)
                              → Fila C (recebe)
```
**Regra:** TODOS que estiverem conectados recebem. A routing key é ignorada.

**Quando usar:** Eventos que precisam ser processados por múltiplos consumidores (como no AlgaSensors).

### Direct

```
Mensagem (key="erro") → [Direct Exchange] → Fila A (binding key="erro")    ✅ recebe
                                           → Fila B (binding key="info")    ❌ não recebe
                                           → Fila C (binding key="erro")    ✅ recebe
```
**Regra:** Só entrega se a routing key da mensagem for IGUAL à binding key da fila.

**Quando usar:** Roteamento específico, como logs por nível (erro, warning, info).

### Topic

```
Mensagem (key="sensor.temperatura.critico") → [Topic Exchange]
  → Fila A (binding="sensor.temperatura.*")     ✅ recebe
  → Fila B (binding="sensor.#")                 ✅ recebe
  → Fila C (binding="sensor.umidade.*")          ❌ não recebe
```
**Regra:** Usa padrões com curingas:
- `*` = substitui exatamente uma palavra
- `#` = substitui zero ou mais palavras

**Quando usar:** Roteamento flexível baseado em categorias hierárquicas.

### Headers

**Regra:** Roteia baseado nos headers da mensagem (não na routing key).

**Quando usar:** Cenários onde a decisão de roteamento depende de múltiplos atributos.

---

## 16. Glossário Visual

```
┌─────────────────────────────────────────────────────────────────────┐
│                         RABBITMQ BROKER                             │
│                                                                     │
│   Producer ────►  EXCHANGE ────binding────► QUEUE ────► Consumer    │
│   (envia)         (roteia)    (conecta)    (armazena)   (recebe)   │
│                                                                     │
│   Tipos:          Tipos:                   Propriedades:            │
│   - Controller    - Fanout (broadcast)     - Durable (persiste)    │
│   - Scheduled     - Direct (exata)         - Exclusive (1 consumidor)│
│   - Service       - Topic (padrão)         - Auto-delete           │
│                   - Headers                                         │
│                                                                     │
│   Classes Spring:                                                   │
│   - RabbitTemplate (enviar)                                         │
│   - @RabbitListener (receber)                                       │
│   - RabbitAdmin (administrar)                                       │
│   - Jackson2JsonMessageConverter (serializar)                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 17. Exercícios Práticos

### Exercício 1 — Observando no Painel (Dificuldade: Fácil)

1. Suba o RabbitMQ com `docker-compose up -d`
2. Acesse `http://localhost:15672` (login: rabbitmq/rabbitmq)
3. Inicie os microsserviços `temperature-processing` e `temperature-monitoring`
4. Envie uma temperatura com curl:
   ```bash
   curl -X POST http://localhost:8081/api/sensors/0HF3YJKPXS9ED/temperatures/data \
     -H "Content-Type: text/plain" -d "25.5"
   ```
5. Observe no painel: a mensagem chegou nas filas? Quantas filas receberam?

### Exercício 2 — Simulando Acúmulo (Dificuldade: Média)

1. **Pare** o `temperature-monitoring` (o consumidor)
2. Envie 10 temperaturas em sequência
3. Vá ao painel e veja as mensagens acumuladas nas filas
4. Agora **inicie** o `temperature-monitoring`
5. Observe as mensagens sendo consumidas gradualmente (por causa do `Thread.sleep(5s)`)

**Reflexão:** O que teria acontecido se fosse comunicação HTTP direta em vez de RabbitMQ?

### Exercício 3 — Criando uma Nova Fila (Dificuldade: Avançada)

Imagine que você precisa de uma nova funcionalidade: registrar TODAS as leituras em um arquivo CSV. Para isso:

1. Crie uma nova constante de fila: `QUEUE_CSV_EXPORT = "temperature-monitoring.csv-export.v1.q"`
2. Crie o `@Bean` para a nova `Queue`
3. Crie o `@Bean` para o novo `Binding` (conecte à mesma exchange Fanout)
4. Crie um novo método com `@RabbitListener` que recebe a mensagem e faz um `log.info()` com os dados

**Resultado esperado:** Agora cada leitura de temperatura será processada por **3** consumidores independentes.

---

## 18. Erros Comuns de Iniciantes

### Erro 1: "Connection refused" na porta 5672

**Causa:** O RabbitMQ não está rodando ou está em outro host.

**Solução:** Verifique com `docker-compose ps` se o container está UP. Certifique-se de que o `host` no `application.yml` está correto.

### Erro 2: Mensagem enviada mas nunca chega na fila

**Causa:** A exchange existe mas não tem nenhum binding. A mensagem é descartada silenciosamente.

**Solução:** Verifique no painel web (aba Exchanges → clique na exchange → veja os bindings). Se não há bindings, o `RabbitMQInitializer` do consumidor pode não ter rodado.

### Erro 3: "Failed to convert message" / erro de deserialização

**Causa:** O consumidor espera um formato de objeto diferente do que o produtor enviou (campos com nomes diferentes, tipos incompatíveis).

**Solução:** Garanta que os campos do objeto no consumidor (`TemperatureLogData`) correspondem exatamente aos campos do produtor (`TemperatureLogOutput`).

### Erro 4: Mensagens sendo reprocessadas infinitamente

**Causa:** O consumidor lança uma exceção, o RabbitMQ recoloca a mensagem na fila, o consumidor tenta de novo, lança exceção de novo...

**Solução:** Implemente uma Dead Letter Queue (DLQ) para mensagens que falharam muitas vezes, ou trate a exceção dentro do listener.

### Erro 5: Apenas um consumidor recebe (quando você quer que todos recebam)

**Causa:** Você está usando a mesma fila para todos os consumidores. Dentro de uma fila, cada mensagem vai para APENAS UM consumidor.

**Solução:** Use filas separadas com bindings para a mesma exchange (como o AlgaSensors faz com `process-temperature.q` e `alerting.q`).

---

## 19. Próximos Passos

Agora que você entende o básico do RabbitMQ com o AlgaSensors, aqui estão os temas para aprofundar:

**Nível Intermediário:**
- Dead Letter Queues (DLQ) — para onde vão mensagens que falharam
- TTL (Time To Live) — mensagens que expiram automaticamente
- Exchanges do tipo Direct e Topic — roteamento mais sofisticado
- Acknowledgment manual — controle fino sobre confirmação de mensagens

**Nível Avançado:**
- Clusters de RabbitMQ — alta disponibilidade
- Quorum Queues — filas replicadas entre nós
- Retry com backoff exponencial — tentativas inteligentes
- Monitoramento com Prometheus e Grafana
- Spring Cloud Stream — abstração de nível mais alto

**Ferramentas para Praticar:**
- Painel web do RabbitMQ (`localhost:15672`) — explore todas as abas
- RabbitMQ Simulator (online) — para visualizar fluxos
- Spring AMQP Reference Documentation — documentação oficial

---

> **Lembre-se:** A melhor forma de aprender RabbitMQ é **experimentando**. Suba o Docker, rode os microsserviços, envie mensagens pelo curl, observe no painel, quebre coisas de propósito e veja o que acontece. Cada erro é uma lição.
>
> Bons estudos!
