# Batch Processing no Mule - GDG DevFest Arapiraca

![MuleSoft](https://img.shields.io/badge/MuleSoft-Batch_Processing-00A0DF?style=flat-square&logo=mulesoft)
![Mule Runtime](https://img.shields.io/badge/Mule_Runtime-4.x-00A0DF?style=flat-square)

## 📋 Sobre o Projeto

Este projeto foi desenvolvido para o **GDG DevFest Arapiraca**, demonstrando conceitos e práticas de **Batch Processing** no MuleSoft Mule Runtime.

## 🎯 O que é Batch Processing?

Batch Processing (processamento em lote) é um recurso exclusivo da **Enterprise Edition** do Mule Runtime que permite processar grandes volumes de dados de forma confiável e assíncrona. É ideal para trabalhar com conjuntos de dados maiores que a memória disponível.

### Características Principais

- **Processamento Assíncrono**: Execução independente do fluxo principal
- **Divisão Automática**: Quebra automática de dados em registros menores
- **Persistência**: Armazena registros em filas persistentes
- **Resiliência**: Retomada automática em caso de falhas ou redeployment
- **Escalabilidade**: Processa grandes volumes sem sobrecarregar a memória

## 🏗️ Arquitetura de Componentes

O Batch Processing no Mule é composto por três componentes principais:

### 1. **Batch Job** (`<batch:job>`)
Componente principal que orquestra todo o processamento em lote:
- Recebe dados de entrada e divide em registros
- Gerencia as fases do processamento
- Gera relatório final com resultados

### 2. **Batch Step** (`<batch:step>`)
Etapas de processamento onde ocorrem as transformações:
- Contém processadores que agem sobre cada registro
- Pode haver múltiplos steps em sequência
- Permite filtragem e roteamento de registros

### 3. **Batch Aggregator** (`<batch:aggregator>`)
Componente opcional para agregação de registros:
- Agrupa registros em arrays
- Útil para inserções em massa
- Apenas um por Batch Step

## 🔄 Fases do Batch Job

### 1. **Load and Dispatch Phase**
- Recebe a mensagem de entrada
- Divide os dados em registros individuais
- Cria uma instância do batch job
- Armazena registros em filas persistentes

### 2. **Process Phase**
- Processa cada registro através dos Batch Steps
- Executa transformações, enriquecimentos e validações
- Propaga variáveis entre steps
- Trata erros de forma individual por registro

### 3. **On Complete Phase**
- Gera relatório final do processamento
- Indica registros com sucesso e falhas
- Disponibiliza métricas da execução
- Permite logging ou notificações

## 📊 Estrutura XML Básica

```xml
<flow name="batch-flow">
  <!-- Trigger do fluxo -->
  <http:listener config-ref="HTTP_Config" path="/batch"/>
  
  <!-- Preparação dos dados -->
  <ee:transform>
    <ee:message>
      <ee:set-payload>
        <![CDATA[%dw 2.0
        output application/json
        ---
        payload.records
        ]]>
      </ee:set-payload>
    </ee:message>
  </ee:transform>
  
  <!-- Batch Job -->
  <batch:job jobName="processRecordsBatch">
    <batch:process-records>
      
      <!-- Primeiro Step: Transformação -->
      <batch:step name="transformStep">
        <ee:transform>
          <ee:message>
            <ee:set-payload>
              <![CDATA[%dw 2.0
              output application/json
              ---
              {
                id: payload.id,
                name: upper(payload.name),
                processedAt: now()
              }
              ]]>
            </ee:set-payload>
          </ee:message>
        </ee:transform>
      </batch:step>
      
      <!-- Segundo Step: Persistência com Agregação -->
      <batch:step name="persistStep">
        <batch:aggregator size="100">
          <database:bulk-insert config-ref="Database_Config">
            <database:sql>
              INSERT INTO customers (id, name, processed_at) 
              VALUES (:id, :name, :processedAt)
            </database:sql>
          </database:bulk-insert>
        </batch:aggregator>
      </batch:step>
      
    </batch:process-records>
    
    <!-- Relatório Final -->
    <batch:on-complete>
      <logger level="INFO" 
              message="Batch completo! Sucessos: #[payload.successfulRecords] | Falhas: #[payload.failedRecords]"/>
    </batch:on-complete>
  </batch:job>
</flow>
```

## 💡 Casos de Uso Comuns

### 1. **Sincronização entre Sistemas**
Sincronizar dados entre aplicações de negócio (ex: Salesforce ↔ NetSuite)

```xml
<batch:job jobName="syncContactsBatch">
  <batch:process-records>
    <batch:step name="getFromSource">
      <salesforce:query config-ref="Salesforce_Config">
        <salesforce:salesforce-query>
          SELECT Id, Name, Email FROM Contact
        </salesforce:salesforce-query>
      </salesforce:query>
    </batch:step>
    
    <batch:step name="upsertToTarget">
      <batch:aggregator size="200">
        <netsuite:upsert-list config-ref="NetSuite_Config"/>
      </batch:aggregator>
    </batch:step>
  </batch:process-records>
</batch:job>
```

### 2. **ETL (Extract, Transform, Load)**
Carregar dados de arquivos CSV para sistemas de destino

```xml
<batch:job jobName="csvToDbBatch">
  <batch:process-records>
    <batch:step name="validateAndTransform">
      <ee:transform>
        <!-- Transformação e validação dos dados -->
      </ee:transform>
    </batch:step>
    
    <batch:step name="loadToDatabase">
      <batch:aggregator size="500">
        <db:bulk-insert config-ref="DB_Config"/>
      </batch:aggregator>
    </batch:step>
  </batch:process-records>
</batch:job>
```

### 3. **Processamento de Dados de API**
Processar grandes volumes de dados de APIs para sistemas legados

## 🎯 Formatos de Entrada Válidos

O Batch Job aceita automaticamente os seguintes formatos:

- ✅ Arrays Java
- ✅ Java Iterables
- ✅ Java Iterators
- ✅ Payloads JSON
- ✅ Payloads XML

> **Importante**: Para outros formatos, transforme os dados antes de entrar no Batch Job usando o Transform Message component.

## 🔐 Propagação de Variáveis

### Durante o Process Phase
- Cada registro herda variáveis do evento de entrada
- Variáveis podem ser modificadas por registro
- Modificações são propagadas entre steps
- Cada registro mantém seu próprio contexto de variáveis

### Durante o On Complete Phase
- Apenas variáveis originais do evento de entrada
- Variáveis criadas nos steps NÃO são propagadas
- Variáveis padrão do batch disponíveis (ex: `batchJobInstanceId`)

## ⚠️ Tratamento de Erros

O Batch Job trata erros a nível de registro individual:

```xml
<batch:step name="processWithErrorHandling" accept-policy="ONLY_FAILURES">
  <logger message="Processando registro com falha: #[payload]"/>
  <!-- Lógica de tratamento ou retry -->
</batch:step>
```

**Accept Policies disponíveis:**
- `ALL` - Todos os registros (padrão)
- `ONLY_FAILURES` - Apenas registros com falha
- `NO_FAILURES` - Apenas registros bem-sucedidos

## 🚀 Performance e Tuning

### Configurações Importantes

```xml
<batch:job jobName="optimizedBatch" maxFailedRecords="100">
  <batch:process-records>
    <batch:step name="processStep">
      <!-- Processadores -->
      
      <batch:aggregator size="1000" streaming="true">
        <!-- Processamento em massa -->
      </batch:aggregator>
    </batch:step>
  </batch:process-records>
</batch:job>
```

**Boas Práticas:**
- Use agregação para inserções em massa
- Configure `maxFailedRecords` para evitar processamento desnecessário
- Utilize streaming para grandes volumes
- Ajuste o tamanho do agregador conforme o destino

## 📚 Recursos Adicionais

- [Documentação Oficial - Batch Processing](https://docs.mulesoft.com/mule-runtime/latest/batch-processing-concept)
- [Batch Job Phases](https://docs.mulesoft.com/mule-runtime/latest/batch-phases)
- [Configuring Batch Components](https://docs.mulesoft.com/mule-runtime/latest/batch-filters-and-batch-aggregator)
- [Handling Errors During Batch Job](https://docs.mulesoft.com/mule-runtime/latest/batch-error-handling-faq)
- [Batch Process Tuning](https://docs.mulesoft.com/mule-runtime/latest/tuning-batch-processing)

## 🛠️ Tecnologias Utilizadas

- MuleSoft Anypoint Studio
- Mule Runtime 4.x (Enterprise Edition)
- DataWeave 2.0
- Maven

## 👥 GDG DevFest Arapiraca

Este projeto faz parte das apresentações do **Google Developers Group DevFest** em Arapiraca, demonstrando boas práticas de integração e processamento de dados em larga escala.

---

**Desenvolvido para GDG DevFest Arapiraca** 🚀

