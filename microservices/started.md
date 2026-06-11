# 🚀 STARTED — Guia Completo: Como Construir Este Projeto do Zero

Este guia ensina passo a passo como **criar cada microserviço do AlgaSensors do zero**, desde a criação do projeto no Spring Initializr até o último arquivo Java. Se você seguir na ordem, ao final terá o sistema funcionando exatamente como o projeto anexado.

---

## 📋 Sumário

1. [Pré-requisitos — O que instalar antes de começar](#1--pré-requisitos--o-que-instalar-antes-de-começar)
2. [Passo 1 — Criar a estrutura raiz e o Docker Compose](#2--passo-1--criar-a-estrutura-raiz-e-o-docker-compose)
3. [Passo 2 — Criar o microserviço device-management](#3--passo-2--criar-o-microserviço-device-management)
4. [Passo 3 — Criar o microserviço temperature-processing](#4--passo-3--criar-o-microserviço-temperature-processing)
5. [Passo 4 — Criar o microserviço temperature-monitoring](#5--passo-4--criar-o-microserviço-temperature-monitoring)
6. [Passo 5 — Executar e testar o sistema completo](#6--passo-5--executar-e-testar-o-sistema-completo)
7. [Troubleshooting — Problemas comuns e soluções](#7--troubleshooting--problemas-comuns-e-soluções)

---

## 1. 🛠 Pré-requisitos — O que instalar antes de começar

Antes de criar qualquer código, você precisa ter instalado na sua máquina:

### 1.1. Java 21 (JDK)

O projeto usa Java 21. Faça o download e instale:

- **Windows/Mac/Linux**: [Adoptium (Eclipse Temurin)](https://adoptium.net/) — clique em "Latest LTS Release" e escolha JDK 21.

Após instalar, verifique no terminal:

```bash
java -version
```

Deve mostrar algo como: `openjdk version "21.x.x"`

### 1.2. Docker e Docker Compose

O RabbitMQ roda em container Docker. Instale:

- **Windows/Mac**: [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- **Linux**: [Docker Engine](https://docs.docker.com/engine/install/) + [Docker Compose Plugin](https://docs.docker.com/compose/install/)

Após instalar, verifique:

```bash
docker --version
docker compose version
```

### 1.3. IDE (Ambiente de Desenvolvimento)

Recomendado: **IntelliJ IDEA Community** (gratuito) — [Download](https://www.jetbrains.com/idea/download/)

Alternativa: **VS Code** com extensões Java (Extension Pack for Java).

### 1.4. cURL ou Postman (para testes)

- **cURL**: Já vem instalado no Mac/Linux. No Windows, use o Git Bash ou instale via [curl.se](https://curl.se/).
- **Postman**: [Download](https://www.postman.com/downloads/) — interface gráfica para testar APIs.

---

## 2. 📦 Passo 1 — Criar a estrutura raiz e o Docker Compose

### 2.1. Criar a pasta raiz do projeto

Crie a seguinte estrutura de pastas no seu computador:

```
algasensors/
├── configs/
│   └── rabbitmq/
├── microservices/
├── docker-compose.yml
└── README.md
```

No terminal:

```bash
mkdir -p algasensors/configs/rabbitmq
mkdir -p algasensors/microservices
cd algasensors
```

### 2.2. Criar o arquivo de plugins do RabbitMQ

Crie o arquivo `configs/rabbitmq/enabled_plugins` com o conteúdo:

```erlang
[rabbitmq_management,rabbitmq_shovel,rabbitmq_shovel_management].
```

**O que cada plugin faz:**
- `rabbitmq_management`: Habilita o painel web de administração (porta 15672).
- `rabbitmq_shovel`: Permite mover mensagens entre filas/brokers.
- `rabbitmq_shovel_management`: Interface gráfica para gerenciar shovels.

### 2.3. Criar o docker-compose.yml

Crie o arquivo `docker-compose.yml` na raiz do projeto:

```yaml
services:
  algasensors-rabbitmq:
    image: rabbitmq:3-management
    restart: no
    ports:
      - 5672:5672
      - 15672:15672
    environment:
      RABBITMQ_DEFAULT_USER: rabbitmq
      RABBITMQ_DEFAULT_PASS: rabbitmq
    volumes:
      - algasensors-rabbitmq:/var/lib/rabbitmq/
      - ./configs/rabbitmq/enabled_plugins:/etc/rabbitmq/enabled_plugins
volumes:
  algasensors-rabbitmq:
```

**Explicação linha a linha:**
- `image: rabbitmq:3-management` — Usa a imagem oficial do RabbitMQ versão 3 com o plugin de management embutido.
- `restart: no` — Não reinicia o container automaticamente se ele parar (ideal para desenvolvimento).
- `ports: 5672:5672` — Expõe a porta AMQP (protocolo de mensageria, usada pelos microserviços para enviar/receber mensagens).
- `ports: 15672:15672` — Expõe a porta do painel de administração web.
- `RABBITMQ_DEFAULT_USER/PASS` — Define as credenciais de acesso (usadas no application.yml dos microserviços).
- `volumes: algasensors-rabbitmq:/var/lib/rabbitmq/` — Persiste os dados do RabbitMQ em um volume Docker (sobrevive a reinicializações do container).
- `volumes: ./configs/rabbitmq/enabled_plugins:...` — Monta o arquivo de plugins dentro do container.

### 2.4. Testar o Docker Compose

```bash
docker compose up -d
```

Acesse http://localhost:15672 — deve aparecer a tela de login do RabbitMQ.
Login: `rabbitmq` / Senha: `rabbitmq`

Se funcionou, o RabbitMQ está pronto. Deixe-o rodando.

---

## 3. 🔧 Passo 2 — Criar o microserviço device-management

Este é o microserviço de **cadastro de sensores**. Vamos criá-lo passo a passo.

### 3.1. Gerar o projeto no Spring Initializr

Acesse [https://start.spring.io](https://start.spring.io) e configure:

| Campo | Valor |
|---|---|
| Project | Gradle - Groovy |
| Language | Java |
| Spring Boot | 3.4.3 |
| Group | com.algaworks.algasensors |
| Artifact | device-management |
| Name | device-management |
| Package name | com.algaworks.algasensors.device.management |
| Packaging | Jar |
| Java | 21 |

**Dependências a adicionar:**
- Spring Web
- Spring Data JPA
- H2 Database
- Lombok

Clique em **Generate** e extraia o ZIP na pasta `algasensors/microservices/`.

### 3.2. Adicionar dependência do TSID no build.gradle

Abra `microservices/device-management/build.gradle` e adicione na seção `dependencies`:

```groovy
implementation 'io.hypersistence:hypersistence-tsid:2.1.4'
```

Também adicione o bloco `idea` (opcional, para download de fontes na IDE):

```groovy
idea {
    module {
        downloadJavadoc = true
        downloadSources = true
    }
}
```

O `build.gradle` final deve ficar assim:

```groovy
plugins {
    id 'idea'
    id 'java'
    id 'org.springframework.boot' version '3.4.3'
    id 'io.spring.dependency-management' version '1.1.7'
}

group = 'com.algaworks.algasensors'
version = '0.0.1-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

idea {
    module {
        downloadJavadoc = true
        downloadSources = true
    }
}

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'io.hypersistence:hypersistence-tsid:2.1.4'
    compileOnly 'org.projectlombok:lombok'
    runtimeOnly 'com.h2database:h2'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

### 3.3. Configurar o application.yml

Renomeie o `application.properties` para `application.yml` (se necessário) em `src/main/resources/` e coloque:

```yaml
server.port: '8080'

spring:
  application.name: device-management
  datasource:
    username: sa
    url: jdbc:h2:file:~/algasensors-device-management-db;CASE_INSENSITIVE_IDENTIFIERS=TRUE;
    driverClassName: org.h2.Driver
    password: '123'
  h2:
    console:
      enabled: 'true'
      settings.web-allow-others: 'true'
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
    hibernate:
      ddl-auto: update
    show-sql: 'true'
```

**Explicação:**
- `server.port: 8080` — Porta HTTP do microserviço.
- `datasource.url: jdbc:h2:file:~/...` — Banco H2 gravado em arquivo na pasta home do usuário. `CASE_INSENSITIVE_IDENTIFIERS=TRUE` faz o H2 ignorar maiúsculas/minúsculas em nomes de tabelas e colunas.
- `h2.console.enabled: true` — Habilita o console web do H2 em `http://localhost:8080/h2-console`.
- `ddl-auto: update` — O Hibernate cria/atualiza tabelas automaticamente com base nas entidades.
- `show-sql: true` — Exibe as queries SQL geradas no console (útil para debug).

### 3.4. Criar os arquivos Java — Camada de Domínio

Agora vamos criar cada arquivo Java na ordem correta. Sempre crie dentro de `src/main/java/com/algaworks/algasensors/device/management/`.

#### 3.4.1. Criar o pacote `common` → `IdGenerator.java`

Caminho: `common/IdGenerator.java`

```java
package com.algaworks.algasensors.device.management.common;

import io.hypersistence.tsid.TSID;
import java.util.Optional;

public class IdGenerator {

    private static final TSID.Factory tsidFactory;

    static {
        Optional.ofNullable(System.getenv("tsid.node"))
                        .ifPresent(tsidNode -> System.setProperty("tsid.node", tsidNode));
        Optional.ofNullable(System.getenv("tsid.node.count"))
                .ifPresent(tsidNodeCount -> System.setProperty("tsid.node.count", tsidNodeCount));
        tsidFactory = TSID.Factory.builder().build();
    }

    private IdGenerator() {}

    public static TSID generateTSID() {
        return tsidFactory.generate();
    }
}
```

**Por que criar isso primeiro?** Porque o IdGenerator é usado na criação de sensores e não depende de nada. Começar pelos utilitários sem dependências é uma boa prática.

#### 3.4.2. Criar o pacote `domain/model` → `SensorId.java`

Caminho: `domain/model/SensorId.java`

```java
package com.algaworks.algasensors.device.management.domain.model;

import io.hypersistence.tsid.TSID;
import jakarta.persistence.Embeddable;
import lombok.AccessLevel;
import lombok.EqualsAndHashCode;
import lombok.Getter;
import lombok.NoArgsConstructor;
import java.io.Serializable;
import java.util.Objects;

@Getter
@Embeddable
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@EqualsAndHashCode
public class SensorId implements Serializable {

    private TSID value;

    public SensorId(TSID value) {
        Objects.requireNonNull(value);
        this.value = value;
    }

    public SensorId(Long value) {
        Objects.requireNonNull(value);
        this.value = TSID.from(value);
    }

    public SensorId(String value) {
        Objects.requireNonNull(value);
        this.value = TSID.from(value);
    }

    @Override
    public String toString() {
        return value.toString();
    }
}
```

**Por que criar isso antes da entidade Sensor?** Porque o `Sensor` depende do `SensorId`. Sempre crie as dependências primeiro.

#### 3.4.3. Criar `domain/model` → `Sensor.java`

Caminho: `domain/model/Sensor.java`

```java
package com.algaworks.algasensors.device.management.domain.model;

import jakarta.persistence.AttributeOverride;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Entity
public class Sensor {
    @Id
    @AttributeOverride(name = "value", column = @Column(name = "id", columnDefinition = "BIGINT"))
    private SensorId id;
    private String name;
    private String ip;
    private String location;
    private String protocol;
    private String model;
    private Boolean enabled;
}
```

#### 3.4.4. Criar `domain/repository` → `SensorRepository.java`

Caminho: `domain/repository/SensorRepository.java`

```java
package com.algaworks.algasensors.device.management.domain.repository;

import com.algaworks.algasensors.device.management.domain.model.Sensor;
import com.algaworks.algasensors.device.management.domain.model.SensorId;
import org.springframework.data.jpa.repository.JpaRepository;

public interface SensorRepository extends JpaRepository<Sensor, SensorId> {
}
```

### 3.5. Criar os arquivos Java — Camada de Configuração (TSID)

Estes arquivos ensinam ao Jackson, JPA e Spring MVC como lidar com o tipo TSID.

#### 3.5.1. `api/config/jackson/TSIDToStringSerializer.java`

```java
package com.algaworks.algasensors.device.management.api.config.jackson;

import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.databind.JsonSerializer;
import com.fasterxml.jackson.databind.SerializerProvider;
import io.hypersistence.tsid.TSID;
import java.io.IOException;

public class TSIDToStringSerializer extends JsonSerializer<TSID> {
    @Override
    public void serialize(TSID value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
        gen.writeString(value.toString());
    }
}
```

#### 3.5.2. `api/config/jackson/StringToTSIDDeserializer.java`

```java
package com.algaworks.algasensors.device.management.api.config.jackson;

import com.fasterxml.jackson.core.JacksonException;
import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.databind.DeserializationContext;
import com.fasterxml.jackson.databind.JsonDeserializer;
import io.hypersistence.tsid.TSID;
import java.io.IOException;

public class StringToTSIDDeserializer extends JsonDeserializer<TSID> {
    @Override
    public TSID deserialize(JsonParser p, DeserializationContext ctxt) throws IOException, JacksonException {
        return TSID.from(p.getText());
    }
}
```

#### 3.5.3. `api/config/jackson/TSIDJacksonConfig.java`

```java
package com.algaworks.algasensors.device.management.api.config.jackson;

import com.fasterxml.jackson.databind.Module;
import com.fasterxml.jackson.databind.module.SimpleModule;
import io.hypersistence.tsid.TSID;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class TSIDJacksonConfig {
    @Bean
    public Module tsidModule() {
        SimpleModule module = new SimpleModule();
        module.addSerializer(TSID.class, new TSIDToStringSerializer());
        module.addDeserializer(TSID.class, new StringToTSIDDeserializer());
        return module;
    }
}
```

#### 3.5.4. `api/config/jpa/TSIDToLongJPAAttributeConverter.java`

```java
package com.algaworks.algasensors.device.management.api.config.jpa;

import io.hypersistence.tsid.TSID;
import jakarta.persistence.AttributeConverter;
import jakarta.persistence.Converter;

@Converter(autoApply = true)
public class TSIDToLongJPAAttributeConverter implements AttributeConverter<TSID, Long> {
    @Override
    public Long convertToDatabaseColumn(TSID attribute) {
        return attribute.toLong();
    }
    @Override
    public TSID convertToEntityAttribute(Long dbData) {
        return TSID.from(dbData);
    }
}
```

#### 3.5.5. `api/config/web/StringToTSIDWebConverter.java`

```java
package com.algaworks.algasensors.device.management.api.config.web;

import io.hypersistence.tsid.TSID;
import org.springframework.core.convert.converter.Converter;

public class StringToTSIDWebConverter implements Converter<String, TSID> {
    @Override
    public TSID convert(String source) {
        return TSID.from(source);
    }
}
```

#### 3.5.6. `api/config/web/WebConfig.java`

```java
package com.algaworks.algasensors.device.management.api.config.web;

import org.springframework.context.annotation.Configuration;
import org.springframework.format.FormatterRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new StringToTSIDWebConverter());
    }
}
```

### 3.6. Criar os arquivos Java — DTOs (Models da API)

#### 3.6.1. `api/model/SensorInput.java`

```java
package com.algaworks.algasensors.device.management.api.model;

import lombok.Data;

@Data
public class SensorInput {
    private String name;
    private String ip;
    private String location;
    private String protocol;
    private String model;
}
```

#### 3.6.2. `api/model/SensorOutput.java`

```java
package com.algaworks.algasensors.device.management.api.model;

import io.hypersistence.tsid.TSID;
import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class SensorOutput {
    private TSID id;
    private String name;
    private String ip;
    private String location;
    private String protocol;
    private String model;
    private Boolean enabled;
}
```

#### 3.6.3. `api/model/SensorMonitoringOuput.java`

```java
package com.algaworks.algasensors.device.management.api.model;

import io.hypersistence.tsid.TSID;
import lombok.Builder;
import lombok.Data;
import java.time.OffsetDateTime;

@Data
@Builder
public class SensorMonitoringOuput {
    private TSID id;
    private Double lastTemperature;
    private OffsetDateTime updatedAt;
    private Boolean enabled;
}
```

#### 3.6.4. `api/model/SensorDetailOutput.java`

```java
package com.algaworks.algasensors.device.management.api.model;

import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class SensorDetailOutput {
    private SensorOutput sensor;
    private SensorMonitoringOuput monitoring;
}
```

### 3.7. Criar os arquivos Java — Cliente HTTP (comunicação entre microserviços)

#### 3.7.1. `api/client/SensorMonitoringClientBadGatewayException.java`

```java
package com.algaworks.algasensors.device.management.api.client;

public class SensorMonitoringClientBadGatewayException extends RuntimeException {
}
```

#### 3.7.2. `api/client/SensorMonitoringClient.java` (Interface)

```java
package com.algaworks.algasensors.device.management.api.client;

import com.algaworks.algasensors.device.management.api.model.SensorMonitoringOuput;
import io.hypersistence.tsid.TSID;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.service.annotation.DeleteExchange;
import org.springframework.web.service.annotation.GetExchange;
import org.springframework.web.service.annotation.HttpExchange;
import org.springframework.web.service.annotation.PutExchange;

@HttpExchange("/api/sensors/{sensorId}/monitoring")
public interface SensorMonitoringClient {

    @PutExchange("/enable")
    void enableMonitoring(@PathVariable TSID sensorId);

    @DeleteExchange("/enable")
    void disableMonitoring(@PathVariable TSID sensorId);

    @GetExchange
    SensorMonitoringOuput getDetail(@PathVariable TSID sensorId);
}
```

#### 3.7.3. `api/client/RestClientFactory.java`

```java
package com.algaworks.algasensors.device.management.api.client;

import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatusCode;
import org.springframework.http.client.ClientHttpRequestFactory;
import org.springframework.http.client.SimpleClientHttpRequestFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;
import java.time.Duration;

@Component
@RequiredArgsConstructor
public class RestClientFactory {

    private final RestClient.Builder builder;

    public RestClient temperatureMonitoringRestClient() {
        return builder.baseUrl("http://localhost:8082")
                .requestFactory(generateClientHttpRequestFactory())
                .defaultStatusHandler(HttpStatusCode::isError, (request, response) -> {
                    throw new SensorMonitoringClientBadGatewayException();
                })
                .build();
    }

    private ClientHttpRequestFactory generateClientHttpRequestFactory() {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        factory.setReadTimeout(Duration.ofSeconds(5));
        factory.setConnectTimeout(Duration.ofSeconds(3));
        return factory;
    }
}
```

#### 3.7.4. `api/client/RestClientConfig.java`

```java
package com.algaworks.algasensors.device.management.api.client;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestClient;
import org.springframework.web.client.support.RestClientAdapter;
import org.springframework.web.service.invoker.HttpServiceProxyFactory;

@Configuration
public class RestClientConfig {

    @Bean
    public SensorMonitoringClient sensorMonitoringClient(RestClientFactory factory) {
        RestClient restClient = factory.temperatureMonitoringRestClient();
        RestClientAdapter adapter = RestClientAdapter.create(restClient);
        HttpServiceProxyFactory proxyFactory = HttpServiceProxyFactory.builderFor(adapter).build();
        return proxyFactory.createClient(SensorMonitoringClient.class);
    }
}
```

#### 3.7.5. `api/client/impl/SensorMonitoringClientImpl.java` (Implementação alternativa)

```java
package com.algaworks.algasensors.device.management.api.client.impl;

import com.algaworks.algasensors.device.management.api.client.RestClientFactory;
import com.algaworks.algasensors.device.management.api.client.SensorMonitoringClient;
import com.algaworks.algasensors.device.management.api.model.SensorMonitoringOuput;
import io.hypersistence.tsid.TSID;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

//@Component
public class SensorMonitoringClientImpl implements SensorMonitoringClient {

    private final RestClient restClient;

    public SensorMonitoringClientImpl(RestClientFactory factory) {
        this.restClient = factory.temperatureMonitoringRestClient();
    }

    @Override
    public void enableMonitoring(TSID sensorId) {
        restClient.put()
                .uri("/api/sensors/{sensorId}/monitoring/enable", sensorId)
                .retrieve()
                .toBodilessEntity();
    }

    @Override
    public void disableMonitoring(TSID sensorId) {
        restClient.delete()
                .uri("/api/sensors/{sensorId}/monitoring/enable", sensorId)
                .retrieve()
                .toBodilessEntity();
    }

    @Override
    public SensorMonitoringOuput getDetail(TSID sensorId) {
        return restClient.get()
                .uri("/api/sensors/{sensorId}/monitoring", sensorId)
                .retrieve()
                .body(SensorMonitoringOuput.class);
    }
}
```

> **IMPORTANTE:** Note que o `@Component` está COMENTADO (`//@Component`). Este arquivo é uma referência de implementação manual. O projeto usa o proxy dinâmico criado pelo `RestClientConfig`.

### 3.8. Criar os arquivos Java — Exception Handler e Controller

#### 3.8.1. `api/config/web/ApiExceptionHandler.java`

```java
package com.algaworks.algasensors.device.management.api.config.web;

import com.algaworks.algasensors.device.management.api.client.SensorMonitoringClientBadGatewayException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler;
import java.io.IOException;
import java.net.ConnectException;
import java.net.SocketTimeoutException;
import java.net.URI;
import java.nio.channels.ClosedChannelException;

@RestControllerAdvice
public class ApiExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler({
            SocketTimeoutException.class,
            ConnectException.class,
            ClosedChannelException.class
    })
    public ProblemDetail handle(IOException e) {
        ProblemDetail problemDetail = ProblemDetail.forStatus(HttpStatus.GATEWAY_TIMEOUT);
        problemDetail.setTitle("Gateway timeout");
        problemDetail.setDetail(e.getMessage());
        problemDetail.setType(URI.create("/errors/gateway-timeout"));
        return problemDetail;
    }

    @ExceptionHandler(SensorMonitoringClientBadGatewayException.class)
    public ProblemDetail handle(SensorMonitoringClientBadGatewayException e) {
        ProblemDetail problemDetail = ProblemDetail.forStatus(HttpStatus.BAD_GATEWAY);
        problemDetail.setTitle("Bad gateway");
        problemDetail.setDetail(e.getMessage());
        problemDetail.setType(URI.create("/errors/bad-gateway"));
        return problemDetail;
    }
}
```

#### 3.8.2. `api/controller/SensorController.java`

```java
package com.algaworks.algasensors.device.management.api.controller;

import com.algaworks.algasensors.device.management.api.client.SensorMonitoringClient;
import com.algaworks.algasensors.device.management.api.model.SensorDetailOutput;
import com.algaworks.algasensors.device.management.api.model.SensorInput;
import com.algaworks.algasensors.device.management.api.model.SensorMonitoringOuput;
import com.algaworks.algasensors.device.management.api.model.SensorOutput;
import com.algaworks.algasensors.device.management.common.IdGenerator;
import com.algaworks.algasensors.device.management.domain.model.Sensor;
import com.algaworks.algasensors.device.management.domain.model.SensorId;
import com.algaworks.algasensors.device.management.domain.repository.SensorRepository;
import io.hypersistence.tsid.TSID;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.server.ResponseStatusException;

@RestController
@RequestMapping("/api/sensors")
@RequiredArgsConstructor
public class SensorController {

    private final SensorRepository sensorRepository;
    private final SensorMonitoringClient sensorMonitoringClient;

    @GetMapping
    public Page<SensorOutput> search(@PageableDefault Pageable pageable) {
        Page<Sensor> sensors = sensorRepository.findAll(pageable);
        return sensors.map(this::convertToModel);
    }

    @GetMapping("{sensorId}")
    public SensorOutput getOne(@PathVariable TSID sensorId) {
        Sensor sensor = sensorRepository.findById(new SensorId(sensorId))
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
        return convertToModel(sensor);
    }

    @GetMapping("{sensorId}/detail")
    public SensorDetailOutput getOneWithDetail(@PathVariable TSID sensorId) {
        Sensor sensor = sensorRepository.findById(new SensorId(sensorId))
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
        SensorMonitoringOuput monitoringOuput = sensorMonitoringClient.getDetail(sensorId);
        SensorOutput sensorOutput = convertToModel(sensor);
        return SensorDetailOutput.builder()
                .monitoring(monitoringOuput)
                .sensor(sensorOutput)
                .build();
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public SensorOutput create(@RequestBody SensorInput input) {
        Sensor sensor = Sensor.builder()
                .id(new SensorId(IdGenerator.generateTSID()))
                .name(input.getName())
                .ip(input.getIp())
                .location(input.getLocation())
                .protocol(input.getProtocol())
                .model(input.getModel())
                .enabled(false)
                .build();
        sensor = sensorRepository.saveAndFlush(sensor);
        return convertToModel(sensor);
    }

    @PutMapping("/{sensorId}")
    public SensorOutput update(@PathVariable TSID sensorId,
                               @RequestBody SensorInput input) {
        Sensor sensor = sensorRepository.findById(new SensorId(sensorId))
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
        sensor.setName(input.getName());
        sensor.setLocation(input.getLocation());
        sensor.setIp(input.getIp());
        sensor.setModel(input.getModel());
        sensor.setProtocol(input.getProtocol());
        sensor = sensorRepository.save(sensor);
        return convertToModel(sensor);
    }

    @DeleteMapping("/{sensorId}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable TSID sensorId) {
        Sensor sensor = sensorRepository.findById(new SensorId(sensorId))
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
        sensorRepository.delete(sensor);
        sensorMonitoringClient.disableMonitoring(sensorId);
    }

    @PutMapping("/{sensorId}/enable")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void enable(@PathVariable TSID sensorId) {
        Sensor sensor = sensorRepository.findById(new SensorId(sensorId))
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
        sensor.setEnabled(true);
        sensorRepository.save(sensor);
        sensorMonitoringClient.enableMonitoring(sensorId);
    }

    @DeleteMapping("/{sensorId}/enable")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void disable(@PathVariable TSID sensorId) {
        Sensor sensor = sensorRepository.findById(new SensorId(sensorId))
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
        sensor.setEnabled(false);
        sensorRepository.save(sensor);
        sensorMonitoringClient.disableMonitoring(sensorId);
    }

    private SensorOutput convertToModel(Sensor sensor) {
        return SensorOutput.builder()
                .id(sensor.getId().getValue())
                .name(sensor.getName())
                .ip(sensor.getIp())
                .location(sensor.getLocation())
                .protocol(sensor.getProtocol())
                .model(sensor.getModel())
                .enabled(sensor.getEnabled())
                .build();
    }
}
```

### 3.9. Testar o device-management

```bash
cd microservices/device-management
./gradlew bootRun
```

Se iniciou sem erros e você vê `Started DeviceManagementApplication`, o primeiro microserviço está pronto!

Teste criando um sensor:

```bash
curl -X POST http://localhost:8080/api/sensors \
  -H "Content-Type: application/json" \
  -d '{"name":"Sensor-01","ip":"10.0.0.1","location":"Sala A","protocol":"MQTT","model":"DHT22"}'
```

---

## 4. 🔧 Passo 3 — Criar o microserviço temperature-processing

Este microserviço **recebe leituras de temperatura e publica no RabbitMQ**.

### 4.1. Gerar o projeto no Spring Initializr

Acesse [https://start.spring.io](https://start.spring.io) com:

| Campo | Valor |
|---|---|
| Group | com.algaworks.algasensors |
| Artifact | temperature-processing |
| Package name | com.algaworks.algasensors.temperature.processing |
| Java | 21 |

**Dependências:**
- Spring Web
- Spring for RabbitMQ
- Lombok

> **ATENÇÃO:** Este microserviço NÃO usa banco de dados (sem JPA, sem H2).

Extraia na pasta `algasensors/microservices/`.

### 4.2. Adicionar dependências extras no build.gradle

Adicione na seção `dependencies`:

```groovy
implementation 'org.apache.commons:commons-lang3:3.17.0'
implementation 'com.fasterxml.uuid:java-uuid-generator:5.1.0'
implementation 'io.hypersistence:hypersistence-tsid:2.1.4'
```

### 4.3. Configurar o application.yml

```yaml
server.port: 8081

spring:
  application.name: temperature-processing
  rabbitmq:
    host: localhost
    port: 5672
    username: rabbitmq
    password: rabbitmq
```

### 4.4. Criar os arquivos Java

Todos os arquivos ficam dentro de `src/main/java/com/algaworks/algasensors/temperature/processing/`.

#### 4.4.1. Configurações TSID (Jackson + Web)

Crie os mesmos arquivos de configuração TSID do device-management, mas com pacotes adaptados. São 4 arquivos:

- `api/config/jackson/TSIDToStringSerializer.java` — Mesmo código, pacote diferente.
- `api/config/jackson/TSIDJacksonConfig.java` — Sem deserializer (só serializa).
- `api/config/web/StringToTSIDWebConverter.java` — Mesmo código, pacote diferente.
- `api/config/web/WebConfig.java` — Mesmo código, pacote diferente.

> **Dica:** No TSIDJacksonConfig deste microserviço, NÃO adicione o deserializer (não há `addDeserializer`), pois este microserviço não recebe TSIDs de JSON — apenas envia.

#### 4.4.2. `common/IdGenerator.java` (UUID v7)

```java
package com.algaworks.algasensors.temperature.processing.common;

import com.fasterxml.uuid.Generators;
import com.fasterxml.uuid.impl.TimeBasedEpochRandomGenerator;
import java.util.UUID;

public class IdGenerator {
    private static final TimeBasedEpochRandomGenerator timeBasedEpochRandomGenerator =
            Generators.timeBasedEpochRandomGenerator();

    private IdGenerator() {}

    public static UUID generateTimeBasedUUID() {
        return timeBasedEpochRandomGenerator.generate();
    }
}
```

#### 4.4.3. `common/UUIDv7Utils.java`

```java
package com.algaworks.algasensors.temperature.processing.common;

import java.time.Instant;
import java.time.OffsetDateTime;
import java.time.ZoneId;
import java.util.UUID;

public class UUIDv7Utils {
    private UUIDv7Utils() {}

    public static OffsetDateTime extractOffsetDateTime(UUID uuid) {
        if (uuid == null) { return null; }
        long timestamp = uuid.getMostSignificantBits() >>> 16;
        return OffsetDateTime.ofInstant(Instant.ofEpochMilli(timestamp), ZoneId.systemDefault());
    }
}
```

#### 4.4.4. `api/model/TemperatureLogOutput.java`

```java
package com.algaworks.algasensors.temperature.processing.api.model;

import io.hypersistence.tsid.TSID;
import lombok.Builder;
import lombok.Data;
import java.time.OffsetDateTime;
import java.util.UUID;

@Data
@Builder
public class TemperatureLogOutput {
    private UUID id;
    private TSID sensorId;
    private OffsetDateTime registeredAt;
    private Double value;
}
```

#### 4.4.5. `infrastructure/rabbitmq/RabbitMQConfig.java`

```java
package com.algaworks.algasensors.temperature.processing.infrastructure.rabbitmq;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.amqp.core.ExchangeBuilder;
import org.springframework.amqp.core.FanoutExchange;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitAdmin;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {

    public static final String FANOUT_EXCHANGE_NAME = "temperature-processing.temperature-received.v1.e";

    @Bean
    public Jackson2JsonMessageConverter jackson2JsonMessageConverter(ObjectMapper objectMapper) {
        return new Jackson2JsonMessageConverter(objectMapper);
    }

    @Bean
    public RabbitAdmin rabbitAdmin(ConnectionFactory connectionFactory) {
        return new RabbitAdmin(connectionFactory);
    }

    @Bean
    public FanoutExchange exchange() {
        return ExchangeBuilder.fanoutExchange(FANOUT_EXCHANGE_NAME).build();
    }
}
```

#### 4.4.6. `infrastructure/rabbitmq/RabbitMQInitializer.java`

```java
package com.algaworks.algasensors.temperature.processing.infrastructure.rabbitmq;

import jakarta.annotation.PostConstruct;
import lombok.RequiredArgsConstructor;
import org.springframework.amqp.rabbit.core.RabbitAdmin;
import org.springframework.context.annotation.Configuration;

@Configuration
@RequiredArgsConstructor
public class RabbitMQInitializer {
    private final RabbitAdmin rabbitAdmin;

    @PostConstruct
    public void init() {
        rabbitAdmin.initialize();
    }
}
```

#### 4.4.7. `api/controller/TemperatureProcessingController.java`

```java
package com.algaworks.algasensors.temperature.processing.api.controller;

import com.algaworks.algasensors.temperature.processing.api.model.TemperatureLogOutput;
import com.algaworks.algasensors.temperature.processing.common.IdGenerator;
import io.hypersistence.tsid.TSID;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.MessagePostProcessor;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.server.ResponseStatusException;
import java.time.OffsetDateTime;

import static com.algaworks.algasensors.temperature.processing.infrastructure.rabbitmq.RabbitMQConfig.FANOUT_EXCHANGE_NAME;

@RestController
@RequestMapping("/api/sensors/{sensorId}/temperatures/data")
@Slf4j
@RequiredArgsConstructor
public class TemperatureProcessingController {

    private final RabbitTemplate rabbitTemplate;

    @PostMapping(consumes = MediaType.TEXT_PLAIN_VALUE)
    public void data(@PathVariable TSID sensorId, @RequestBody String input) {
        if (input == null || input.isBlank()) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST);
        }
        Double temperature;
        try {
            temperature = Double.parseDouble(input);
        } catch (NumberFormatException e) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST);
        }

        TemperatureLogOutput logOutput = TemperatureLogOutput.builder()
                .id(IdGenerator.generateTimeBasedUUID())
                .sensorId(sensorId)
                .value(temperature)
                .registeredAt(OffsetDateTime.now())
                .build();

        log.info(logOutput.toString());

        String exchange = FANOUT_EXCHANGE_NAME;
        String routingKey = "";
        Object payload = logOutput;

        MessagePostProcessor messagePostProcessor = message -> {
            message.getMessageProperties().setHeader("sensorId", logOutput.getSensorId().toString());
            return message;
        };
        rabbitTemplate.convertAndSend(exchange, routingKey, payload, messagePostProcessor);
    }
}
```

### 4.5. Testar o temperature-processing

```bash
cd microservices/temperature-processing
./gradlew bootRun
```

Se iniciou sem erros, está pronto! Mas as mensagens ainda não têm consumidor — vamos criá-lo agora.

---

## 5. 🔧 Passo 4 — Criar o microserviço temperature-monitoring

Este é o microserviço **mais complexo**. Ele consome mensagens do RabbitMQ, persiste dados e expõe APIs.

### 5.1. Gerar o projeto no Spring Initializr

| Campo | Valor |
|---|---|
| Group | com.algaworks.algasensors |
| Artifact | temperature-monitoring |
| Package name | com.algaworks.algasensors.temperature.monitoring |
| Java | 21 |

**Dependências:**
- Spring Web
- Spring Data JPA
- H2 Database
- Spring for RabbitMQ
- Lombok

Extraia em `algasensors/microservices/`.

### 5.2. Adicionar dependência do TSID

No `build.gradle`, adicione:

```groovy
implementation 'io.hypersistence:hypersistence-tsid:2.1.4'
```

### 5.3. Configurar o application.yml

```yaml
server:
  port: '8082'

spring:
  application.name: temperature-monitoring
  datasource:
    username: sa
    url: jdbc:h2:file:~/algasensors-temperature-monitoring-db;CASE_INSENSITIVE_IDENTIFIERS=TRUE;AUTO_SERVER=TRUE
    driverClassName: org.h2.Driver
    password: '123'
  h2:
    console:
      enabled: 'true'
      settings.web-allow-others: 'true'
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
    hibernate:
      ddl-auto: update
    show-sql: 'false'
  rabbitmq:
    host: localhost
    port: 5672
    username: rabbitmq
    password: rabbitmq
    listener:
      simple:
        prefetch: 4
        retry:
          enabled: true
          initial-interval: 10s
          max-interval: 20s
          multiplier: 2
          max-attempts: 3
```

**Configurações importantes do RabbitMQ:**
- `prefetch: 4` — Cada consumidor pega no máximo 4 mensagens de uma vez.
- `retry.enabled: true` — Ativa reprocessamento automático de mensagens que falharem.
- `retry.max-attempts: 3` — No máximo 3 tentativas antes de enviar para a DLQ.
- `retry.initial-interval: 10s` — Espera 10 segundos entre a 1ª e 2ª tentativa.
- `retry.multiplier: 2` — O intervalo dobra a cada tentativa (10s → 20s).
- `retry.max-interval: 20s` — O intervalo nunca ultrapassa 20 segundos.

### 5.4. Criar os arquivos Java — Ordem de criação

Todos dentro de `src/main/java/com/algaworks/algasensors/temperature/monitoring/`.

A ordem recomendada de criação é:

**1) Configurações TSID** (mesma lógica dos outros microserviços, 6 arquivos):
- `api/config/jackson/TSIDToStringSerializer.java`
- `api/config/jackson/StringToTSIDDeserializer.java`
- `api/config/jackson/TSIDJacksonConfig.java` (com serializer E deserializer)
- `api/config/jpa/TSIDToLongJPAAttributeConverter.java`
- `api/config/web/StringToTSIDWebConverter.java`
- `api/config/web/WebConfig.java`

**2) Domínio — Value Objects e Entidades:**
- `domain/model/SensorId.java` (mesmo do device-management, pacote diferente)
- `domain/model/TemperatureLogId.java`
- `domain/model/SensorMonitoring.java`
- `domain/model/TemperatureLog.java`
- `domain/model/SensorAlert.java`

**3) Domínio — Repositórios:**
- `domain/repository/SensorMonitoringRepository.java`
- `domain/repository/SensorAlertRepository.java`
- `domain/repository/TemperatureLogRepository.java`

**4) DTOs (Models da API):**
- `api/model/TemperatureLogData.java`
- `api/model/SensorMonitoringOuput.java`
- `api/model/SensorAlertInput.java`
- `api/model/SensorAlertOutput.java`

**5) Serviços de domínio:**
- `domain/service/TemperatureMonitoringService.java`
- `domain/service/SensorAlertService.java`

**6) Infraestrutura RabbitMQ:**
- `infrastructure/rabbitmq/RabbitMQConfig.java`
- `infrastructure/rabbitmq/RabbitMQInitializer.java`
- `infrastructure/rabbitmq/RabbitMQListener.java`

**7) Controllers:**
- `api/controller/SensorMonitoringController.java`
- `api/controller/SensorAlertController.java`
- `api/controller/TemperatureLogController.java`

> **IMPORTANTE:** O conteúdo de cada arquivo está no projeto comentado (algasensors-comentado.zip). Copie o código de lá respeitando os pacotes. Todos os arquivos deste microserviço já foram listados integralmente no README.md e comentados no zip anterior.

### 5.5. Testar o temperature-monitoring

```bash
cd microservices/temperature-monitoring
./gradlew bootRun
```

Se iniciou sem erros e você vê no log algo como `Created new connection: rabbitConnectionFactory`, o consumidor está conectado ao RabbitMQ!

---

## 6. ✅ Passo 5 — Executar e testar o sistema completo

### 6.1. Checklist de inicialização

Certifique-se de que tudo está rodando (em terminais separados):

| Serviço | Comando | Porta |
|---|---|---|
| RabbitMQ | `docker compose up -d` | 5672 / 15672 |
| device-management | `./gradlew bootRun` | 8080 |
| temperature-processing | `./gradlew bootRun` | 8081 |
| temperature-monitoring | `./gradlew bootRun` | 8082 |

### 6.2. Teste completo passo a passo

**Passo A — Criar um sensor:**

```bash
curl -s -X POST http://localhost:8080/api/sensors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Sensor Estufa",
    "ip": "192.168.1.50",
    "location": "Estufa Principal",
    "protocol": "MQTT",
    "model": "DHT22"
  }'
```

Copie o valor do campo `"id"` da resposta (ex: `"0HPWDZA0GJM5P"`).

**Passo B — Ativar o sensor:**

```bash
curl -X PUT http://localhost:8080/api/sensors/SEU_ID_AQUI/enable
```

Isso faz DUAS coisas: ativa no device-management e chama o temperature-monitoring para ativar o monitoramento.

**Passo C — Configurar alerta de temperatura:**

```bash
curl -X PUT http://localhost:8082/api/sensors/SEU_ID_AQUI/alert \
  -H "Content-Type: application/json" \
  -d '{"maxTemperature": 35.0, "minTemperature": 10.0}'
```

**Passo D — Enviar leituras de temperatura:**

```bash
# Temperatura normal (dentro dos limites)
curl -X POST http://localhost:8081/api/sensors/SEU_ID_AQUI/temperatures/data \
  -H "Content-Type: text/plain" -d "22.5"

# Temperatura alta (acima do máximo) — vai gerar alerta!
curl -X POST http://localhost:8081/api/sensors/SEU_ID_AQUI/temperatures/data \
  -H "Content-Type: text/plain" -d "38.0"

# Temperatura baixa (abaixo do mínimo) — vai gerar alerta!
curl -X POST http://localhost:8081/api/sensors/SEU_ID_AQUI/temperatures/data \
  -H "Content-Type: text/plain" -d "5.0"
```

**Passo E — Verificar os resultados:**

```bash
# Ver última temperatura e status do monitoramento
curl -s http://localhost:8082/api/sensors/SEU_ID_AQUI/monitoring

# Ver histórico de temperaturas
curl -s http://localhost:8082/api/sensors/SEU_ID_AQUI/temperatures

# Ver dados completos (sensor + monitoramento)
curl -s http://localhost:8080/api/sensors/SEU_ID_AQUI/detail
```

**Passo F — Verificar os logs no terminal do temperature-monitoring:**

Você deve ver mensagens como:
```
Temperature Updated: SensorId 0HPWDZA0GJM5P Temp 22.5
Alert Ignored: SensorId 0HPWDZA0GJM5P Temp 22.5
Temperature Updated: SensorId 0HPWDZA0GJM5P Temp 38.0
Alert Max Temp: SensorId 0HPWDZA0GJM5P Temp 38.0
Temperature Updated: SensorId 0HPWDZA0GJM5P Temp 5.0
Alert Min Temp: SensorId 0HPWDZA0GJM5P Temp 5.0
```

**Passo G — Verificar o RabbitMQ:**

Acesse http://localhost:15672 (rabbitmq/rabbitmq) e vá na aba "Queues":
- `temperature-monitoring.process-temperature.v1.q` — Fila de processamento.
- `temperature-monitoring.alerting.v1.q` — Fila de alertas.
- `temperature-monitoring.process-temperature.v1.dlq` — Dead letter queue (deve estar vazia se tudo deu certo).

### 6.3. Verificar os bancos H2

- **device-management**: http://localhost:8080/h2-console
  - JDBC URL: `jdbc:h2:file:~/algasensors-device-management-db`
  - User: `sa` / Password: `123`
  
- **temperature-monitoring**: http://localhost:8082/h2-console
  - JDBC URL: `jdbc:h2:file:~/algasensors-temperature-monitoring-db`
  - User: `sa` / Password: `123`

---

## 7. 🔥 Troubleshooting — Problemas comuns e soluções

### "Port 8080 already in use"
Outro programa está usando a porta. Mate o processo ou altere a porta no `application.yml`.

```bash
# Linux/Mac: encontrar o processo
lsof -i :8080
# Windows:
netstat -ano | findstr :8080
```

### "Connection refused" ao chamar temperature-monitoring
O microserviço temperature-monitoring (porta 8082) não está rodando. Inicie-o primeiro com `./gradlew bootRun`.

### "Cannot connect to RabbitMQ" / "Connection refused: localhost:5672"
O RabbitMQ não está rodando. Verifique com `docker compose ps` e inicie com `docker compose up -d`.

### "Table not found" no H2
Verifique se o `ddl-auto` está como `update` no `application.yml`. Se o problema persistir, delete os arquivos do banco na pasta home (`~/algasensors-*-db.*`) e reinicie.

### Mensagens ficam na fila e não são consumidas
Verifique se o temperature-monitoring está rodando e conectado ao RabbitMQ. Procure no log por `Created new connection: rabbitConnectionFactory`.

### Mensagens vão para a DLQ (Dead Letter Queue)
Verifique o log do temperature-monitoring para entender a exceção. As causas mais comuns são: erro de deserialização JSON (campos com nomes diferentes) ou erro no banco de dados.

### Gradle não encontra o Java 21
Certifique-se de que o `JAVA_HOME` aponta para o JDK 21:
```bash
echo $JAVA_HOME
java -version
```

### A IDE não reconhece os getters/setters do Lombok
Instale o plugin Lombok na IDE:
- **IntelliJ**: Settings → Plugins → buscar "Lombok" → Install
- **VS Code**: A extensão "Extension Pack for Java" já inclui suporte a Lombok.

Após instalar, reinicie a IDE e faça "Refresh Gradle" no projeto.

---

## 📊 Resumo da ordem de criação

| Ordem | O que criar | Por quê nesta ordem? |
|---|---|---|
| 1 | Pasta raiz + Docker Compose | Infraestrutura (RabbitMQ) precisa estar pronta antes |
| 2 | device-management | É independente (não depende de RabbitMQ) |
| 3 | temperature-processing | Produz mensagens (precisa do RabbitMQ rodando) |
| 4 | temperature-monitoring | Consome mensagens (precisa do producer e do RabbitMQ) |

Dentro de cada microserviço, a ordem de criação dos arquivos segue a regra: **crie primeiro o que não depende de nada** (utilitários, value objects, entidades) e por último o que depende de tudo (controllers, listeners).

---

**Pronto!** Seguindo este guia do início ao fim, você terá construído um sistema completo de microserviços com Spring Boot, RabbitMQ, H2, TSID e comunicação assíncrona.
