# 📦 Sistema de Integração de Pedidos com Apache Camel

## 📋 Visão Geral

Este projeto implementa um sistema de integração de pedidos utilizando o framework **Apache Camel** para processamento e roteamento de pedidos de uma livraria. O sistema demonstra padrões de integração empresarial (EIP - Enterprise Integration Patterns) para conectar diferentes sistemas através de mensageria JMS, serviços REST e SOAP.

> **Projeto Educacional**: Desenvolvido com base no curso "Apache Camel: O Framework de Integração entre Sistemas" da **DevMedia**.

## 🎯 Objetivo do Projeto

Demonstrar a capacidade do Apache Camel de:
- Integrar múltiplos sistemas heterogêneos
- Processar pedidos de forma assíncrona
- Transformar mensagens entre diferentes formatos
- Implementar tratamento robusto de erros e resiliência
- Aplicar padrões de integração empresarial

## 🏗️ Arquitetura

### Fluxo de Integração

```
┌─────────────────┐
│  Arquivo XML    │
│  (pedidos/)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  ActiveMQ       │
│  (JMS Queue)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Validação XSD  │
└────────┬────────┘
         │
         ├─────────────────┬──────────────────┐
         ▼                 ▼                  ▼
   ┌──────────┐      ┌──────────┐      ┌──────────┐
   │  Filtro  │      │   XSLT   │      │   HTTP   │
   │  EBOOK   │      │Transform.│      │  REST    │
   └────┬─────┘      └────┬─────┘      └──────────┘
        │                 │
        ▼                 ▼
   ┌──────────┐      ┌──────────┐
   │ REST API │      │   SOAP   │
   │  (eBook) │      │Financeiro│
   └──────────┘      └──────────┘
```

### Componentes Principais

#### 1. **RotaEnviaPedidos.java**
**Responsabilidade**: Produtor de mensagens
- Monitora o diretório `pedidos/` em busca de novos arquivos XML
- Envia automaticamente os pedidos para a fila JMS do ActiveMQ
- Utiliza o componente `file:` do Camel para polling de arquivos

**Endpoint de origem**: `file:pedidos?noop=true`
**Endpoint de destino**: `activemq:queue:pedidos`

#### 2. **RotaPedidos.java**
**Responsabilidade**: Consumidor e orquestrador de integração
- Consome mensagens da fila JMS
- Valida pedidos contra schema XSD
- Distribui mensagens para múltiplos destinos (multicast)
- Processa rotas específicas para cada tipo de integração

### Rotas Implementadas

#### 🔹 Rota Principal (`rota-pedidos`)
```java
from("activemq:queue:pedidos")
    .to("validator:pedido.xsd")
    .multicast()
        .to("direct:soap")
        .to("direct:http")
```
- **Validação**: Garante conformidade com schema XSD
- **Multicast**: Distribui mensagem para processamento paralelo

#### 🔹 Rota HTTP (`rota-http`)
Processa pedidos de **eBooks** e envia para API REST:
- **Extração**: XPath para extrair IDs (pedido, cliente, livro)
- **Split**: Separa itens do pedido
- **Filtro**: Processa apenas itens com `formato=EBOOK`
- **Transformação**: XML → JSON (marshal)
- **Endpoint**: `http://localhost:8080/webservices/ebook/item`

```java
from("direct:http")
    .setProperty("pedidoId", xpath("/pedido/id/text()"))
    .setProperty("clienteId", xpath("/pedido/pagamento/email-titular/text()"))
    .split().xpath("/pedido/itens/item")
    .filter().xpath("/item/formato[text()='EBOOK']")
    .setProperty("ebookId", xpath("/item/livro/codigo/text()"))
    .marshal().xmljson()
    .setHeader(Exchange.HTTP_METHOD, HttpMethods.GET)
    .to("http4://localhost:8080/webservices/ebook/item")
```

#### 🔹 Rota SOAP (`rota-soap`)
Integra com serviço financeiro via SOAP:
- **Transformação**: XSLT (XML → SOAP Envelope)
- **Endpoint**: `http://localhost:8080/webservices/financeiro`

```java
from("direct:soap")
    .to("xslt:pedido-para-soap.xslt")
    .setHeader(Exchange.CONTENT_TYPE, constant("text/xml"))
    .to("http4://localhost:8080/webservices/financeiro")
```

## 🛠️ Tecnologias e Recursos

### Framework e Bibliotecas

| Tecnologia | Versão | Propósito |
|------------|--------|-----------|
| **Apache Camel** | 2.16.1 | Framework de integração e roteamento |
| **ActiveMQ** | 5.6.0 | Message broker JMS |
| **Camel HTTP4** | 2.16.1 | Cliente HTTP para REST/SOAP |
| **Camel JMS** | 2.16.1 | Integração com messaging |
| **Camel XMLJson** | 2.16.1 | Conversão XML ↔ JSON |
| **XOM** | 1.2.5 | Processamento XML |
| **Log4j** | 2.4 | Sistema de logs |
| **SLF4J** | 1.7.12 | Abstração de logging |

### Componentes Apache Camel Utilizados

- **`file:`** - Monitoramento de sistema de arquivos
- **`activemq:`** - Mensageria JMS
- **`validator:`** - Validação contra XSD
- **`direct:`** - Comunicação síncrona entre rotas
- **`http4:`** - Requisições HTTP/REST/SOAP
- **`xslt:`** - Transformação XSLT
- **`xpath:`** - Extração de dados XML

### Padrões de Integração Empresarial (EIP)

#### 1. **Message Channel**
Utilização de filas JMS para comunicação assíncrona:
```
activemq:queue:pedidos
activemq:queue:pedidos.DLQ (Dead Letter Queue)
```

#### 2. **Content-Based Router**
Filtragem de itens por tipo (EBOOK vs IMPRESSO):
```java
.filter().xpath("/item/formato[text()='EBOOK']")
```

#### 3. **Message Translator**
Transformações de formato:
- XML → JSON (marshal)
- XML → SOAP Envelope (XSLT)

#### 4. **Splitter**
Separação de pedidos em itens individuais:
```java
.split().xpath("/pedido/itens/item")
```

#### 5. **Multicast**
Distribuição paralela para múltiplos destinos:
```java
.multicast()
    .to("direct:soap")
    .to("direct:http")
```

#### 6. **Dead Letter Channel**
Tratamento de mensagens com falha:
```java
errorHandler(deadLetterChannel("activemq:queue:pedidos.DLQ")
    .maximumRedeliveries(3)
    .redeliveryDelay(2000)
    .logExhaustedMessageHistory(true))
```

## 🔄 Tratamento de Erros e Resiliência

### Estratégia de Error Handler

O projeto implementa um robusto mecanismo de tratamento de erros:

```java
errorHandler(deadLetterChannel("activemq:queue:pedidos.DLQ")
    .logExhaustedMessageHistory(true)
    .maximumRedeliveries(3)
    .redeliveryDelay(2000)
    .onRedelivery(new Processor() {
        public void process(Exchange exchange) {
            int counter = exchange.getIn().getHeader(Exchange.REDELIVERY_COUNTER);
            int max = exchange.getIn().getHeader(Exchange.REDELIVERY_MAX_COUNTER);
            System.out.println("Redelivery " + counter + "/" + max);
        }
    })
)
```

**Características**:
- ✅ **Tentativas automáticas**: 3 redeliveries
- ✅ **Delay exponencial**: 2 segundos entre tentativas
- ✅ **Dead Letter Queue**: Mensagens problemáticas isoladas
- ✅ **Logging**: Histórico completo de tentativas
- ✅ **Monitoramento**: Contador de redeliveries

## 📁 Estrutura do Projeto

```
apache-camel-main/
│
├── src/
│   └── main/
│       ├── java/
│       │   └── br/com/caelum/camel/
│       │       ├── RotaEnviaPedidos.java    # Produtor JMS
│       │       └── RotaPedidos.java          # Consumidor e orquestrador
│       │
│       └── resources/
│           ├── pedido.xsd                    # Schema de validação
│           ├── pedido-para-soap.xslt         # Transformação SOAP
│           └── log4j2.xml                    # Configuração de logs
│
├── pedidos/                                  # Diretório monitorado
│   ├── 1_pedido.xml
│   ├── 2_pedido.xml
│   ├── 3_pedido.xml
│   └── 4_pedido.xml
│
├── pom.xml                                   # Dependências Maven
├── .gitignore
└── README.md
```

## 📄 Modelo de Dados

### Estrutura do Pedido (XML)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<pedido>
    <id>2451256</id>
    <dataCompra>2013-12-05T18:21:07.529-02:00</dataCompra>
    <itens>
        <item>
            <formato>EBOOK</formato>
            <quantidade>1</quantidade>
            <livro>
                <codigo>ARQ</codigo>
                <titulo>Introdução à Arquitetura e Design de Software</titulo>
                <tituloCurto>Arquitetura Java</tituloCurto>
                <nomeAutor>Sergio Lopes, Paulo Silveira, ...</nomeAutor>
                <valorEbook>29.90</valorEbook>
                <valorImpresso>79.90</valorImpresso>
            </livro>
        </item>
    </itens>
    <pagamento>
        <status>CONFIRMADO</status>
        <valor>29.90</valor>
        <titular>Edgar Brião</titular>
        <email-titular>edgar.b@abc.com</email-titular>
    </pagamento>
</pedido>
```

### Schema XSD

O arquivo `pedido.xsd` define:
- ✅ Validação de tipos de dados
- ✅ Campos obrigatórios e opcionais
- ✅ Enumerações (status: CONFIRMADO/CANCELADO)
- ✅ Formatos (EBOOK/IMPRESSO)
- ✅ Tipos customizados (positive_decimal_type)

## 🚀 Como Executar

### Pré-requisitos

1. **Java JDK 8** ou superior
2. **Apache Maven 3.x**
3. **Apache ActiveMQ** 5.6.0+
4. **Servidor de aplicação** com webservices configurados

### Configuração do ActiveMQ

```bash
# Download e extração
wget https://archive.apache.org/dist/activemq/5.6.0/apache-activemq-5.6.0-bin.tar.gz
tar -xzf apache-activemq-5.6.0-bin.tar.gz

# Iniciar ActiveMQ
cd apache-activemq-5.6.0/bin
./activemq start

# Console web disponível em:
# http://localhost:8161/admin (admin/admin)
```

### Compilação

```bash
cd apache-camel-main
mvn clean install
```

### Execução

#### 1. Iniciar o Consumidor (RotaPedidos)
```bash
mvn exec:java -Dexec.mainClass="br.com.caelum.camel.RotaPedidos"
```

#### 2. Iniciar o Produtor (RotaEnviaPedidos)
```bash
mvn exec:java -Dexec.mainClass="br.com.caelum.camel.RotaEnviaPedidos"
```

### Testando o Sistema

1. Coloque arquivos XML no diretório `pedidos/`
2. Monitore os logs para ver o processamento
3. Verifique o console do ActiveMQ para status das filas
4. Confira os endpoints REST/SOAP para validar integrações

## 📊 Monitoramento

### Logs

O sistema utiliza Log4j2 para rastreamento:
```
- Processamento de rotas
- Tentativas de redelivery
- Erros e exceções
- Transformações de mensagens
```

### ActiveMQ Console

Acesse `http://localhost:8161/admin` para:
- Visualizar mensagens nas filas
- Monitorar Dead Letter Queue
- Verificar throughput e performance
- Gerenciar conexões

## 🎓 Conceitos Aprendidos

Este projeto demonstra os seguintes tópicos do curso:

### ✅ Fundamentos
- Criação de rotas Camel
- Configuração de endpoints
- Integração com sistema de arquivos

### ✅ Processamento de Mensagens
- Separação de mensagens (Splitter)
- Filtragem baseada em conteúdo (Content-Based Router)
- Validação com XSD Schema

### ✅ Conectividade
- Endpoints HTTP/REST
- Integração SOAP
- Mensageria JMS com ActiveMQ

### ✅ Transformação
- XPath para extração de dados
- XSLT para transformação XML
- Marshal XML → JSON

### ✅ Padrões Avançados
- Sub-rotas com `direct:`
- Multicast para distribuição paralela
- Error Handling e Dead Letter Channel
- Redelivery policies

### ✅ Boas Práticas
- Separação de responsabilidades
- Tratamento robusto de erros
- Logging e rastreabilidade
- Configuração externalizada

## 🔧 Possíveis Melhorias

### Sugestões de Evolução

1. **Persistência de Dados**
   - Integrar com banco de dados usando `camel-sql`
   - Armazenar histórico de pedidos

2. **Segurança**
   - Implementar autenticação nos endpoints
   - Criptografia de mensagens sensíveis

3. **Monitoramento Avançado**
   - Integração com Prometheus/Grafana
   - Métricas de performance

4. **Escalabilidade**
   - Configuração de thread pools
   - Load balancing de consumidores

5. **Testes**
   - Testes unitários com `camel-test`
   - Testes de integração com ActiveMQ embedded

6. **Containerização**
   - Dockerfile para aplicação
   - Docker Compose com ActiveMQ

## 📚 Recursos Adicionais

### Documentação Oficial
- [Apache Camel Documentation](https://camel.apache.org/manual/)
- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/)
- [ActiveMQ Documentation](https://activemq.apache.org/)

### Padrões de Integração
- Message Channel
- Content-Based Router
- Message Translator
- Splitter
- Multicast
- Dead Letter Channel

## 📝 Notas Importantes

- ⚠️ **Produção**: Este é um projeto educacional. Para produção, considere:
  - Configuração externalizada (properties/YAML)
  - Autenticação e autorização
  - Monitoramento enterprise
  - Testes abrangentes
  - Alta disponibilidade

- ⏱️ **Timeouts**: As rotas executam por 20 segundos (`Thread.sleep(20000)`)
- 🔄 **Modo noop**: O componente file usa `noop=true` (não move/deleta arquivos)
- 🎯 **Endpoints**: URLs hardcoded - considere externalizar para configuração

## 👨‍💻 Autor

Projeto desenvolvido como parte do curso **"Apache Camel: O Framework de Integração entre Sistemas"** da **DevMedia**.

## 📄 Licença

Projeto educacional para fins de aprendizado.

---

**Desenvolvido com Apache Camel** 🐪 | **DevMedia Course** 📚
