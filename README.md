# 🚀 Central de Observabilidade (Stack OTel + LGTM)

Este repositório contém uma stack de observabilidade completa e centralizada, pronta para monitorar múltiplas aplicações. A arquitetura é baseada no padrão OTel (OpenTelemetry) e na "LGTM Stack" (Loki, Grafana, Tempo, Mimir/Prometheus).

O objetivo é ter um **único local** que recebe, armazena e visualiza:
* **Métricas** (Prometheus)
* **Logs** (Loki)
* **Traces** (Tempo)

## 🏛️ Arquitetura da Solução

Esta stack é composta por 6 serviços principais orquestrados via `docker-compose.yml`:

1.  **Grafana:** A plataforma de visualização e dashboard onde você verá todos os seus dados.
2.  **Prometheus:** O banco de dados de séries temporais para armazenar **Métricas**.
3.  **Loki:** O sistema de armazenamento para **Logs**.
4.  **Tempo:** O sistema de armazenamento para **Traces** (rastreamentos).
5.  **Promtail:** O agente coletor de logs para **Infraestrutura**. Ele "assiste" os logs do Docker e os envia para o Loki.
6.  **OTel Collector (Contrib):** O "porteiro" principal para suas **Aplicações**. Ele recebe telemetria (logs, traces, métricas) via protocolo OTLP e os distribui para o destino correto (Loki, Tempo ou Prometheus).

## ⚙️ Estrutura de Arquivos

Central-Monitoramento/ ├── docker-compose.yml # Orquestrador principal dos serviços ├── prometheus/ │ └── prometheus.yml # Define o que o Prometheus deve "raspar" (scrape) ├── promtail/ │ └── promtail-config.yml # Configura o Promtail para descobrir logs do Docker ├── tempo/ │ └── tempo-config.yml # Configuração do armazém de traces └── otel-collector/ └── otel-config.yml # Configura as "esteiras" (pipelines) de telemetria


## ▶️ Como Executar

**Pré-requisitos:**
* Docker
* Docker Compose

1.  Clone este repositório.
2.  Navegue até a pasta raiz do projeto no seu terminal.
3.  Execute o comando para iniciar todos os serviços em segundo plano:
    ```bash
    docker-compose up -d
    ```
4.  Para parar todos os serviços, execute:
    ```bash
    docker-compose down
    ```

## 🔌 Acessando os Serviços

Uma vez que os containers estão no ar (`up`), você pode acessar as interfaces:

* **🖥️ Grafana (Painel Principal):**
    * **URL:** `http://localhost:3000`
    * **Login:** `admin` / `admin` (será solicitado trocar a senha no primeiro acesso)

* **🎯 Prometheus (Métricas):**
    * **URL:** `http://localhost:9090`

* **📄 Loki (Logs):**
    * **URL:** `http://localhost:3100`

* **📍 OTel Collector (Ponto de Entrada para Apps):**
    * **URL (HTTP):** `http://localhost:4318`
    * **URL (gRPC):** `http://localhost:4317`
    * *Este é o endereço que suas aplicações (frontend/backend) devem usar para enviar dados OTLP.*

### Configuração Inicial do Grafana (Data Sources)

Após o primeiro login no Grafana, você precisa conectar as fontes de dados.

1.  Acesse `http://localhost:3000`.
2.  No menu lateral (⚙️), vá em **"Connections"** > **"Data Sources"**.
3.  Clique em **"Add new data source"** para cada um dos serviços abaixo.

> **IMPORTANTE:** Como o Grafana está rodando dentro da mesma rede Docker Compose, use o **nome do serviço** (ex: `http://loki:3100`), e não `localhost`.

* **Fonte 1: Prometheus**
    * **Tipo:** `Prometheus`
    * **URL:** `http://prometheus:9090`
    * Clique em "Save & Test".

* **Fonte 2: Loki**
    * **Tipo:** `Loki`
    * **URL:** `http://loki:3100`
    * Clique em "Save & Test".

* **Fonte 3: Tempo**
    * **Tipo:** `Tempo`
    * **URL:** `http://tempo:3200`
    * Clique em "Save & Test".

## ⚠️ Observações e Pontos de Atenção (Troubleshooting)

Esta seção documenta os problemas mais comuns encontrados durante a configuração desta stack.

---

### 1. `Connection was aborted` ou `connect: connection refused` (App -> OTel -> Tempo)

* **Problema:** Ao enviar um *trace* (via `curl` ou aplicação), o cliente recebe `(56) Recv failure: Connection was aborted`. Ao verificar os logs do `otel_collector`, vemos um erro `dial tcp 172...:4317: connect: connection refused`.
* **Causa:** O `otel-collector` está a receber os dados, mas não consegue entregá-los ao `tempo`. O log do `tempo` mostra que ele está a escutar apenas em `127.0.0.1:4317` (localhost *interno* do seu *container*), recusando ligações da rede Docker.
* **Solução:** Dizer explicitamente ao `tempo` para escutar em **todas** as interfaces (`0.0.0.0`) no `tempo-config.yml`.

    ```yaml
    # No /tempo/tempo-config.yml
    distributor:
      receivers:
        otlp:
          protocols:
            grpc:
              endpoint: 0.0.0.0:4317 # <-- A correção
            http:
              endpoint: 0.0.0.0:4318 # <-- A correção
    ```

### 2. Erro de `CORS` no Frontend (Browser -> OTel Collector)

* **Problema:** A aplicação *frontend* (ex: `localhost:5173`) falha ao enviar *traces* para o *collector* (ex: `localhost:4318`), e o console do navegador mostra um erro de `CORS`.
* **Causa:** O navegador bloqueia, por segurança, que um site (`localhost:5173`) faça requisições para um domínio/porta diferente (`localhost:4318`), a menos que o destino permita.
* **Solução:** Adicionar uma configuração de `cors` no `otel-config.yml` para permitir requisições vindas da origem do seu *frontend*.

    ```yaml
    # No /otel-collector/otel-config.yml
    receivers:
      otlp: 
        protocols:
          http:
            # ...
            cors:
              allowed_origins: 
                - "http://localhost:5173" # <-- Permite o seu frontend
              allowed_headers: ["*"]
    ```

### 3. Erro `permission denied` no Volume do Tempo

* **Problema:** O *container* `tempo` falha ao iniciar ou reinicia em *loop*. Os logs mostram um erro `permission denied` ao tentar escrever em `/tmp/tempo/traces`.
* **Causa:** O utilizador que corre o processo dentro do *container* `tempo` não tem permissão para escrever no volume montado pelo Docker.
* **Solução (Desenvolvimento):** Forçar o *container* a correr como `root` (utilizador `0`), que tem permissão total.

    ```yaml
    # No docker-compose.yml
    services:
      tempo:
        # ...
        user: "0" # <-- A correção
        volumes:
          - ./tempo/tempo-config.yml:/etc/tempo.yml
          - tempo-data:/tmp/tempo/traces
    ```

### 4. Exportador Loki no OTel Collector (A Mudança Crítica)

* **Problema:** O `otel-collector` falhava ao tentar usar o exportador `loki:`.
* **Causa:** A imagem `otel/opentelemetry-collector-contrib:latest` removeu o exportador `loki` nativo.
* **Solução (Correta):** O Loki agora aceita dados diretamente via OTLP. A configuração correta no `otel-config.yml` é usar o exportador `otlphttp:` apontando para o *endpoint* OTLP do Loki.

    *No `otel-config.yml` (seção `exporters`):*
    ```yaml
    exporters:
      otlphttp/loki:
        endpoint: "http://loki:3100/otlp"
    ```
    *E na pipeline de logs:*
    ```yaml
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp/loki] # <-- Usando o novo exportador
    ```

### 5. Erro `no such file or directory` (Configuração de Volumes)

* **Problema:** Os *containers* `tempo` ou `otel-collector` falhavam ao iniciar com um erro de "arquivo não encontrado".
* **Causa:** No arquivo `docker-compose.yml`, o caminho na diretiva `command:` (que diz ao *container* qual arquivo ler) e o caminho de destino na diretiva `volumes:` (que "injeta" o arquivo) não eram idênticos.
* **Solução:** Garanta que os caminhos são os mesmos.

    *Exemplo (Tempo):*
    ```yaml
    command: [-config.file=/etc/tempo.yml]  # <-- Este caminho
    volumes:
      - ./tempo/tempo-config.yml:/etc/tempo.yml # <-- Deve ser idêntico a este
    ```

### 6. Erro `yaml: unmarshal errors` (Configuração do Tempo)

* **Problema:** O *container* `tempo` iniciava, mas falhava ao ler o `tempo-config.yml` devido a um erro de formato.
* **Causa:** A imagem `grafana/tempo:latest` mudou. Campos de configuração (como `ingester:`) podem se tornar obsoletos ou ter a indentação incorreta.
* **Solução:** Usar a estrutura correta para a versão (ver o `tempo-config.yml` deste repositório), que foca em definir os *endpoints* dentro de `distributor.receivers`.

### 7. Erro de "Ciclo de Reinício" (`received SIGINT/SIGTERM`)

* **Problema:** O *container* `tempo` subia com sucesso, mas era "desligado" (via `SIGTERM`) após alguns segundos, entrando em um *loop* de reinicialização.
* **Causa:** O `tempo` estava saudável, mas o `otel-collector`, que tinha um `depends_on: [tempo]`, estava falhando ao iniciar (crashando). O Docker Compose, ao ver o `otel-collector` falhar, desligava os serviços relacionados (incluindo o `tempo`) para tentar reiniciar a *stack*.
* **Lição:** Se um *container* "saudável" continua reiniciando, **verifique os logs dos *containers* que dependem dele**.

### 8. Configuração de `Resource` na instrumentação (TypeScript)

* **Problema:** Ao usar `new Resource()` em um projeto TypeScript (especialmente com `verbatimModuleSyntax` ativado), o compilador pode gerar erros (`ts(1484)`), tratando `Resource` como um tipo e não como uma classe instanciável.
* **Solução (Correta):** A forma recomendada é usar as funções de fábrica `defaultResource` e `resourceFromAttributes` e combiná-las. Isso captura os atributos padrões do ambiente (ex: navegador) e os mescla com os seus atributos personalizados (ex: nome do serviço).

    *Exemplo de importação:*
    ```javascript
    import { resourceFromAttributes, defaultResource } from '@opentelemetry/resources';
    import { ATTR_SERVICE_NAME } from '@opentelemetry/semantic-conventions';
    ```
    *Exemplo de uso:*
    ```javascript
    const myResource = defaultResource().merge(
      resourceFromAttributes({
        [ATTR_SERVICE_NAME]: 'meu-frontend',
        // ... outros atributos
      })
    );
    ```