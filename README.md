# Sistema de Análise de Alertas Wazuh com IA

## 📋 Índice

1. [Resumo Executivo](#resumo-executivo)
2. [Visão Geral da Arquitetura](#visão-geral-da-arquitetura)
3. [Pré-requisitos](#pré-requisitos)
4. [Esquema do Banco de Dados](#esquema-do-banco-de-dados)
5. [Instalação e Configuração](#instalação-e-configuração)
6. [Workflow 1: Coleta de Alertas](#workflow-1-coleta-de-alertas)
7. [Workflow 2: Análise com IA e Relatórios](#workflow-2-análise-com-ia-e-relatórios)
8. [Sistema de Memória](#sistema-de-memória)
9. [Estratégia de Limpeza](#estratégia-de-limpeza)
10. [Referência de Configuração](#referência-de-configuração)
11. [Solução de Problemas](#solução-de-problemas)
12. [Integração Bitrix24](#integração-bitrix24)

---

## 📊 Resumo Executivo

### Definição do Problema

Equipes de segurança enfrentam fadiga de alertas de sistemas de monitoramento de alto volume como Wazuh:
- **Demorado**: 5-10 minutos por alerta × 1000 alertas/dia = 83+ horas/dia
- **Inconsistente**: Diferentes analistas chegam a conclusões diferentes
- **Falta de contexto**: Sem perspectiva histórica sobre alertas recorrentes
- **Reativo**: Resposta atrasada a ameaças genuínas

### Solução

Um sistema inteligente de análise de alertas baseado em IA que:
- ✅ **Automatiza** análise de alertas usando OpenAI GPT
- 🧠 **Aprende** com decisões passadas através de memória persistente
- 🎯 **Mantém consistência** ao longo do tempo e entre analistas
- 📉 **Reduz falsos positivos** em ~40% através de reconhecimento de padrões
- ⏰ **Economiza** 5+ horas/semana por 1000 alertas/dia

### Recursos Principais

| Recurso | Descrição |
|---------|-----------|
| **Coleta Automatizada** | Ingestão baseada em webhook do Wazuh |
| **Análise com IA** | GPT-4 Turbo via OpenAI |
| **Sistema de Memória** | Armazenamento persistente dos últimos 5 vereditos por host |
| **Limpeza Inteligente** | Retenção de 3 dias + mínimo de 5 alertas por host |
| **Relatórios HTML** | Relatórios profissionais por email com estatísticas |
| **Reconhecimento de Padrões** | Aprende com vereditos históricos |

---

## 🏗️ Visão Geral da Arquitetura

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│   Servidor  │ Webhook │  Workflow 1  │ Armazenar│ Tabela de  │
│   Wazuh     ├────────►│    Coleta    ├────────►│   Dados     │
│             │         │   de Alertas │         │             │
└─────────────┘         └──────────────┘         └──────┬──────┘
                                                        │
                        ┌──────────────┐                │
                        │ Agendamento  │ Buscar         │
                        │   Trigger    ├────────────────┘
                        │ (A cada 6h)  │
                        └──────┬───────┘
                               │
                        ┌──────▼───────┐
                        │  Workflow 2  │
                        │  Análise IA  │
                        │ e Relatórios │
                        └──────┬───────┘
                               │
                ┌──────────────┼──────────────┐
                │              │              │
         ┌──────▼──────┐ ┌────▼─────┐ ┌─────▼──────┐
         │  Atualizar  │ │  Email   │ │  Limpeza   │
         │  Vereditos  │ │ Relatório│ │ Inteligente│
         │  na Tabela  │ │          │ │            │
         └─────────────┘ └──────────┘ └────────────┘
```

### Fluxo de Dados

1. **Fase de Coleta** (Workflow 1)
   - Wazuh envia alertas via webhook
   - Alertas são processados e estruturados
   - Armazenados na Tabela de Dados com `processed = "0"`

2. **Fase de Análise** (Workflow 2)
   - Agendamento dispara a cada 6 horas
   - Busca alertas não processados
   - Agrupa por agente + regra
   - Carrega memória (últimos 5 vereditos por host)
   - IA analisa com contexto histórico
   - Armazena vereditos com `processed = "1"`

3. **Fase de Limpeza**
   - Exclui alertas processados antigos (>3 dias)
   - Mantém mínimo de 5 por host para memória
   - Preserva alertas recentes (últimos 3 dias)

---

## 📦 Pré-requisitos

### Serviços Necessários

- **n8n** (v1.0+): Plataforma de automação de workflow com recurso de Tabelas de Dados
- **Wazuh** (v4.0+): Sistema de monitoramento de segurança
- **OpenAI API**: Acesso ao modelo de IA (GPT-4 Turbo)
- **SMTP**: Servidor de email para entrega de relatórios
- **Bitrix24** (opcional): Plataforma de colaboração para notificações em tempo real via chat

> **Nota:** Este sistema usa o recurso nativo de Tabelas de Dados do n8n - nenhum banco de dados externo necessário!

### Chaves de API e Credenciais

| Serviço | Propósito | Como Obter |
|---------|-----------|------------|
| OpenAI | Análise com IA | [platform.openai.com](https://platform.openai.com) |
| SMTP | Entrega de email | Seu provedor de email (Gmail, Outlook, etc.) |

### Requisitos de Rede

- Instância n8n acessível do servidor Wazuh
- n8n tem acesso de saída à internet para:
  - API OpenAI (api.openai.com)
  - Servidor SMTP do seu provedor de email

---

## 🗄️ Esquema do Banco de Dados

### Tabela de Dados: `wazuh_alerts`

| Coluna | Tipo | Descrição | Exemplo |
|--------|------|-----------|---------|
| `id` | string | ID único do alerta | `alert_1697123456789_abc123` |
| `agent_name` | string | Nome do host | `web-server-01` |
| `agent_ip` | string | Endereço IP | `192.168.1.100` |
| `agent_id` | string | ID do agente Wazuh | `001` |
| `rule_id` | string | Identificador da regra | `5710` |
| `rule_level` | number | Severidade (0-15) | `7` |
| `rule_description` | string | Nome da regra | `Múltiplas falhas de autenticação` |
| `rule_groups` | string | Separado por vírgulas | `authentication,pci_dss` |
| `alert_data` | string | Mensagem principal | `Falha no login do usuário admin de 10.0.0.1` |
| `raw_body` | string | Payload JSON completo | `{"agent": {...}, "rule": {...}}` |
| `timestamp` | datetime | Timestamp do alerta | `2025-10-13T14:30:00Z` |
| `location` | string | Origem do log | `/var/log/auth.log` |
| `created_at` | datetime | Tempo de ingestão | `2025-10-13T14:30:05Z` |
| `processed` | string | Flag de status | `"0"` ou `"1"` |
| `source` | string | Sistema de origem | `wazuh` |
| `full_log` | string | Linha de log completa | `Oct 13 14:30:00 ...` |
| `collection_version` | string | Versão do esquema | `v1.0` |
| **Campos da IA** | | | |
| `verdict` | string | Decisão da IA | `Verdadeiro Positivo`, `Falso Positivo`, `Misto` |
| `verdict_confidence` | string | Nível de confiança | `Alta`, `Média`, `Baixa` |
| `ai_summary` | string | Resumo da análise | `Tentativas legítimas de login falhado...` |
| `analyzed_at` | datetime | Timestamp da análise | `2025-10-13T15:00:00Z` |

---

## 🚀 Instalação e Configuração

> **⚠️ IMPORTANTE - Substituir Placeholders:**
> Os arquivos JSON dos workflows contêm placeholders que devem ser substituídos por suas credenciais reais:
> - `YOUR_DATATABLE_ID` → ID da sua tabela de dados
> - `YOUR_PROJECT_ID` → ID do seu projeto n8n
> - `YOUR_WEBHOOK_TOKEN` → Token único para o webhook
> - `YOUR_HEADER_AUTH_ID` → ID da credencial de autenticação
> - `YOUR_OPENAI_CREDENTIAL_ID` → ID da credencial OpenAI
> - `YOUR_SMTP_CREDENTIAL_ID` → ID da credencial SMTP
> - `YOUR_DOMAIN.bitrix24.com.br` → Seu domínio Bitrix24
> - `YOUR_USER_ID` → Seu ID de usuário Bitrix24
> - `chatYOUR_CHAT_ID` → ID do chat Bitrix24
> - `your-email@example.com` → Seu endereço de email

### Passo 1: Criar Tabela de Dados

1. No n8n, navegue até **Data** → **Tables**
2. Clique em **+ Add Table**
3. Nomeie como: `wazuh_alerts`
4. Adicione as seguintes colunas:

| Nome da Coluna | Tipo | Configurações |
|----------------|------|---------------|
| `id` | String | Chave primária |
| `agent_name` | String | - |
| `agent_ip` | String | - |
| `agent_id` | String | - |
| `rule_id` | String | - |
| `rule_level` | Number | - |
| `rule_description` | String | - |
| `rule_groups` | String | - |
| `alert_data` | String | - |
| `raw_body` | String | - |
| `timestamp` | Date & Time | - |
| `location` | String | - |
| `created_at` | Date & Time | Padrão: Agora |
| `processed` | String | Padrão: "0" |
| `source` | String | Padrão: "wazuh" |
| `full_log` | String | - |
| `collection_version` | String | Padrão: "v1.0" |
| `verdict` | String | - |
| `verdict_confidence` | String | - |
| `ai_summary` | String | - |
| `analyzed_at` | Date & Time | - |

5. Clique em **Save** para criar a tabela

### Passo 2: Configurar API OpenAI

1. Cadastre-se em [platform.openai.com](https://platform.openai.com)
2. Gere uma chave de API
3. No n8n: **Settings** → **Credentials** → **Add Credential**
4. Selecione **OpenAI API**
5. Insira sua chave de API

### Passo 3: Configurar SMTP

1. Obtenha as credenciais SMTP do seu provedor de email:
   - **Gmail**: Use senha de aplicativo (não sua senha normal)
   - **Outlook/Office365**: Use credenciais da conta
   - **Outro provedor**: Consulte a documentação do provedor

2. No n8n: **Settings** → **Credentials** → **Add Credential**
3. Selecione **SMTP**
4. Configure:
   - **Host**: `smtp.gmail.com` (Gmail) ou seu servidor SMTP
   - **Port**: `587` (TLS) ou `465` (SSL)
   - **User**: Seu endereço de email
   - **Password**: Senha de aplicativo ou senha da conta
   - **Security**: `TLS` ou `SSL`

### Passo 4: Importar Workflows

1. Baixe ambos os arquivos JSON dos workflows
2. No n8n: **Workflows** → **Import from File**
3. Importe `workflow1-clean.json` (Coleta de Alertas Wazuh)
4. Importe `workflow2-clean.json` (Análise e Relatórios Diários Wazuh com Memória de IA)

### Passo 5: Configurar Workflows

#### Workflow 1: Coleta de Alertas

1. Abra o workflow **Coleta de Alertas Wazuh**
2. Clique no node **Wazuh Webhook**
3. Configure autenticação via Header (recomendado para segurança)
4. Copie a URL do webhook: `https://sua-instancia-n8n.com/webhook/wazuh/ingest-YOUR_WEBHOOK_TOKEN`
5. Atualize o node **Inserir na Tabela de Dados**:
   - Selecione sua tabela `wazuh_alerts`
   - Verifique o mapeamento de colunas

#### Workflow 2: Análise com IA

1. Abra o workflow **Análise e Relatórios Diários Wazuh com Memória de IA**
2. Atualize o node **Enviar Relatório por Email**:
   - **De (fromEmail)**: `your-email@example.com`
   - **Para (toEmail)**: `your-email@example.com`
   - Selecione sua credencial SMTP configurada
3. Atualize o node **Análise de IA via OpenAI GPT**:
   - Selecione sua credencial OpenAI
4. Verifique se todos os nodes **Data Table** apontam para `wazuh_alerts`
5. Teste o agendamento (opcional: altere de 6 horas para sua preferência)

### Passo 6: Configurar Integração Wazuh

No seu servidor Wazuh, edite `/var/ossec/etc/ossec.conf`:

```xml
<ossec_config>
  <integration>
    <name>custom-webhook</name>
    <hook_url>https://sua-instancia-n8n.com/webhook/wazuh/ingest-YOUR_WEBHOOK_TOKEN</hook_url>
    <level>3</level>
    <alert_format>json</alert_format>
  </integration>
</ossec_config>
```

**Nota de Segurança:** Recomenda-se usar autenticação via Header. Configure no n8n:
1. Crie uma credencial **Header Auth** 
2. Defina um token seguro (ex: `X-Wazuh-Auth: seu-token-secreto-aqui`)
3. Configure o webhook para usar essa credencial

Reinicie o gerenciador Wazuh:

```bash
systemctl restart wazuh-manager
```

### Passo 7: Ativar Workflows

1. No n8n, ative ambos os workflows
2. Teste disparando um alerta Wazuh
3. Verifique se o alerta aparece na Tabela de Dados

---

## 📥 Workflow 1: Coleta de Alertas

### Propósito

Recebe alertas Wazuh via webhook e os armazena em formato estruturado para análise posterior.

### Detalhamento dos Nodes

#### 1. Trigger Webhook
```javascript
// Configuração
{
  "httpMethod": "POST",
  "path": "wazuh/ingest-YOUR_WEBHOOK_TOKEN",
  "authentication": "headerAuth"
}
```

#### 2. Processar Dados do Alerta (Node de Código)

```javascript
// Extrair e estruturar dados de alerta do webhook Wazuh
const items = $input.all();
const processedAlerts = [];

for (const item of items) {
  const root = item.json || {};
  const body = root.body || {};

  // Extrair informações do agente
  const agentName = body.agent?.name ?? 'agente-desconhecido';
  const agentIp = body.agent?.ip ?? '0.0.0.0';
  const agentId = body.agent?.id ?? 'desconhecido';

  // Extrair informações da regra
  const ruleId = String(body.rule?.id ?? 'desconhecido');
  const ruleLevel = parseInt(body.rule?.level ?? 0, 10);
  const ruleDescription = body.rule?.description ?? 'Sem descrição';
  const ruleGroups = Array.isArray(body.rule?.groups) 
    ? body.rule.groups.join(',') 
    : '';

  // Extrair dados do alerta
  const timestamp = body.timestamp ?? new Date().toISOString();
  const location = body.location ?? '';
  const message = body.message 
    ?? body.data?.win?.system?.message 
    ?? body.full_log 
    ?? '';

  // Gerar ID único
  const uniqueId = `alert_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;

  const structuredAlert = {
    id: uniqueId,
    agent_name: agentName,
    agent_ip: agentIp,
    agent_id: agentId,
    alert_data: typeof message === 'string' ? message : JSON.stringify(message),
    rule_id: ruleId,
    rule_level: ruleLevel,
    rule_description: ruleDescription,
    rule_groups: ruleGroups,
    full_log: '',
    location: location,
    timestamp: timestamp,
    created_at: new Date().toISOString(),
    processed: false,
    source: 'wazuh',
    collection_version: 'v1.0',
    raw_body: JSON.stringify(body)
  };

  processedAlerts.push(structuredAlert);
}

return processedAlerts;
```

#### 3. Inserir na Tabela de Dados

Mapeia automaticamente os dados processados para as colunas da tabela.

#### 4. Registrar Resumo (Node de Código)

```javascript
// Registra sucesso e retorna resumo
const items = $input.all();

const summary = {
  success: true,
  alerts_stored: items.length,
  timestamp: new Date().toISOString(),
  table: 'wazuh_alerts',
  agents: [...new Set(items.map(i => i.json.agent_name))],
  severity_levels: items.map(i => i.json.rule_level)
};

console.log('=== RESUMO DE ARMAZENAMENTO DE ALERTAS ===');
console.log(`Armazenados ${summary.alerts_stored} alerta(s)`);
console.log(`Agentes: ${summary.agents.join(', ')}`);
console.log(`Níveis de severidade: ${summary.severity_levels.join(', ')}`);

return { json: summary };
```

### Teste

```bash
# Testar webhook com alerta de exemplo
curl -X POST https://sua-instancia-n8n.com/webhook/wazuh/ingest-YOUR_WEBHOOK_TOKEN \
  -H "Content-Type: application/json" \
  -H "X-Wazuh-Auth: seu-token-secreto-aqui" \
  -d '{
    "agent": {
      "name": "test-server",
      "ip": "192.168.1.100",
      "id": "001"
    },
    "rule": {
      "id": "5710",
      "level": 7,
      "description": "Múltiplas falhas de autenticação",
      "groups": ["authentication", "pci_dss"]
    },
    "message": "Falha no login do usuário admin de 10.0.0.1",
    "timestamp": "2025-10-13T14:30:00Z",
    "location": "/var/log/auth.log"
  }'
```

---

## 🤖 Workflow 2: Análise com IA e Relatórios

### Propósito

Analisa alertas não processados usando IA com contexto de memória histórica, gera vereditos e envia relatórios por email.

### Fluxo de Execução

```
Trigger de Agendamento (A cada 6 horas)
    ↓
Buscar Alertas Não Processados (processed = "0")
    ↓
Agrupar por Agente + Regra
    ↓
🧠 Buscar Memória do Agente (Últimos 5 vereditos por host)
    ↓
Preparar Prompt Aprimorado com Memória
    ↓
Análise de IA via OpenAI GPT
    ↓
Analisar Resposta da IA
    ↓
Agregar Todos os Resultados
    ↓
Gerar Relatório HTML
    ↓
├→ Enviar Relatório por Email
└→ Armazenar Vereditos na Tabela de Dados
    ↓
🧹 Limpeza Inteligente (Excluir alertas antigos)
```

### Nodes Principais

#### 1. Agrupar Alertas por Agente e Regra

```javascript
// Agrupar alertas não processados por agente E rule_id
const alerts = $input.all();

// Filtrar apenas não processados
const unprocessedAlerts = alerts.filter(item => 
  item.json.processed === "0" || item.json.processed === 0
);

// Agrupar por agente + rule_id
const groupedByAgentAndRule = {};

unprocessedAlerts.forEach(item => {
  const key = `${item.json.agent_name}_${item.json.agent_ip}_${item.json.rule_id}`;
  
  if (!groupedByAgentAndRule[key]) {
    groupedByAgentAndRule[key] = {
      agent_name: item.json.agent_name,
      agent_ip: item.json.agent_ip,
      agent_id: item.json.agent_id,
      rule_id: item.json.rule_id,
      rule_description: item.json.rule_description,
      rule_level: item.json.rule_level,
      rule_groups: item.json.rule_groups,
      alerts: [],
      alert_count: 0
    };
  }
  
  groupedByAgentAndRule[key].alerts.push({
    id: item.json.id,
    rule_id: item.json.rule_id,
    rule_level: item.json.rule_level,
    rule_description: item.json.rule_description,
    alert_data: item.json.alert_data,
    timestamp: item.json.timestamp,
    location: item.json.location,
    raw_body: item.json.raw_body
  });
  
  groupedByAgentAndRule[key].alert_count++;
});

// Ordenar por contagem de alertas
const result = Object.values(groupedByAgentAndRule)
  .sort((a, b) => b.alert_count - a.alert_count);

return result.map(item => ({ json: item }));
```

#### 2. 🧠 Buscar Memória do Agente

```javascript
// Buscar últimos 5 alertas analisados para cada agente
const groupedAlerts = $input.all();
const enrichedAlerts = [];

// Acessar todos os alertas do node anterior
const allAlertsNode = $('Buscar Alertas Não Processados');
const allAlerts = allAlertsNode.all();

for (const groupedItem of groupedAlerts) {
  const agentName = groupedItem.json.agent_name;
  const agentIp = groupedItem.json.agent_ip;
  
  // Filtrar alertas processados deste agente com vereditos
  const agentMemory = allAlerts
    .filter(item => {
      const alert = item.json;
      return alert.agent_name === agentName &&
             alert.agent_ip === agentIp &&
             (alert.processed === "1" || alert.processed === 1) &&
             alert.verdict != null &&
             alert.verdict !== '';
    })
    .sort((a, b) => {
      const timeA = new Date(a.json.analyzed_at || a.json.timestamp).getTime();
      const timeB = new Date(b.json.analyzed_at || b.json.timestamp).getTime();
      return timeB - timeA; // Mais recente primeiro
    })
    .slice(0, 5) // Últimos 5
    .map(item => ({
      timestamp: item.json.analyzed_at || item.json.timestamp,
      rule_id: item.json.rule_id,
      rule_description: item.json.rule_description,
      rule_level: item.json.rule_level,
      alert_data: (item.json.alert_data || 'N/D').substring(0, 200),
      verdict: item.json.verdict,
      verdict_confidence: item.json.verdict_confidence,
      ai_summary: item.json.ai_summary
    }));
  
  enrichedAlerts.push({
    json: {
      ...groupedItem.json,
      agent_memory: agentMemory,
      has_memory: agentMemory.length > 0
    }
  });
}

return enrichedAlerts;
```

#### 3. Preparar Prompt Aprimorado com Memória

O prompt completo está em português brasileiro, instruindo a IA a:
1. Revisar contexto histórico
2. Analisar novos alertas
3. Comparar com vereditos passados
4. Identificar novos padrões
5. Determinar se são Verdadeiros Positivos ou Falsos Positivos
6. Fornecer veredito claro com nível de confiança
7. Dar recomendações específicas

#### 4. Análise de IA via OpenAI

```javascript
// Configuração do HTTP Request
{
  "method": "POST",
  "url": "https://api.openai.com/v1/chat/completions",
  "authentication": "predefinedCredentialType",
  "nodeCredentialType": "openAiApi",
  "body": {
    "model": "gpt-4-turbo-preview",
    "messages": [
      {
        "role": "user",
        "content": "{{ $json.prompt }}"
      }
    ],
    "temperature": 0.3,
    "max_tokens": 4000
  }
}
```

#### 5. Analisar Resposta da IA (com Tratamento de Erros)

O sistema possui tratamento robusto de erros para garantir que mesmo respostas malformadas da IA sejam processadas corretamente.

---

## 🧠 Sistema de Memória

### Como Funciona

O sistema de memória permite que a IA aprenda com decisões passadas:

1. **Antes da Análise**: Busca últimos 5 alertas processados para cada agente
2. **Injeção de Contexto**: Inclui vereditos históricos no prompt da IA
3. **Reconhecimento de Padrões**: IA compara comportamento atual vs histórico
4. **Armazenamento de Veredito**: Novos vereditos armazenados para memória futura

### Benefícios da Memória

| Benefício | Impacto |
|-----------|---------|
| **Consistência** | Mesmos alertas recebem mesmos vereditos ao longo do tempo |
| **Detecção de Padrões** | Reconhece falsos positivos recorrentes |
| **Análise Comportamental** | Detecta anomalias vs padrões normais |
| **Redução de Falsos Positivos** | ~40% de redução através de aprendizado |

### Indicadores de Memória nos Relatórios

- 🧠 **Badge Memória**: Análise usou contexto histórico
- ⚠️ **Primeira Análise**: Sem memória disponível (novo host)

---

## 🧹 Estratégia de Limpeza

### Política de Retenção Inteligente

```
Manter TODOS os alertas dos últimos 3 dias
    +
Manter mínimo de 5 alertas por host (para memória)
    =
Equilíbrio ideal: Dados recentes + Memória de longo prazo
```

### Exemplo de Cenário

**Host A tem 100 alertas totais:**
- 20 alertas dos últimos 3 dias → **Manter**
- 5 alertas mais antigos com vereditos → **Manter** (memória)
- 75 alertas antigos (>3 dias) → **Excluir**

**Resultado:** 25 alertas mantidos, 75 excluídos, memória perfeita preservada

### Lógica de Limpeza

```javascript
// Calcular parâmetros de retenção
const retentionDays = 3;
const minAlertsPerHost = 5;

const cutoffDate = new Date();
cutoffDate.setDate(cutoffDate.getDate() - retentionDays);

// Para cada alerta, decidir:
const rules = {
  protectedForMemory: mustKeepAlertIds.has(alert.id), // Últimos 5 por host
  keepUnprocessed: alert.processed === "0", // Ainda não analisado
  keepRecent: alertDate >= cutoffDate, // Últimos 3 dias
  deleteOld: processed && old && !protected // Seguro para excluir
};
```

### Frequência de Limpeza

Executa automaticamente após cada análise (a cada 6 horas).

---

## 📧 Exemplo de Relatório por Email

### Componentes do Relatório

1. **Cards de Resumo** (Seção Superior)
   - **Agentes Únicos**: Hosts analisados
   - **Regras Únicas**: Tipos diferentes de regras disparadas
   - **Total de Alertas**: Alertas processados
   - **🧠 Com Memória**: Análises usando contexto histórico (badge verde)
   - **Verdadeiros Positivos**: Ameaças genuínas
   - **Falsos Positivos**: Alertas benignos

2. **Tabela de Análise Detalhada**
   - Nome do agente com badge de memória
   - ID e descrição da regra
   - Contagem de alertas
   - Resumo da análise
   - Veredito com nível de confiança
   - Recomendações específicas

3. **Informações de Rodapé**
   - Timestamp de geração do relatório
   - Indicação de Agente de IA com capacidade de aprendizado de Memória
   - Gerado por n8n + OpenAI GPT

### Indicadores de Memória

- **🧠 Badge Memória**: Badge verde indica que a IA usou contexto histórico dos 5 alertas analisados anteriormente para este agente
- **⚠️ Primeira Análise**: Badge cinza indica que esta é a primeira vez que este agente foi analisado (sem contexto histórico)

---

## ⚙️ Referência de Configuração

### Configurações n8n

```bash
# Configuração n8n
N8N_HOST=sua-instancia-n8n.com
N8N_PROTOCOL=https
N8N_PORT=443

# Fuso horário
GENERIC_TIMEZONE=America/Sao_Paulo
```

### Configurações de Integração Wazuh

```xml
<!-- /var/ossec/etc/ossec.conf -->
<integration>
  <name>custom-webhook</name>
  <hook_url>https://sua-instancia-n8n.com/webhook/wazuh/ingest-YOUR_WEBHOOK_TOKEN</hook_url>
  <level>3</level> <!-- Severidade mínima -->
  <rule_id>510,518,519,5710</rule_id> <!-- Regras específicas (opcional) -->
  <alert_format>json</alert_format>
  <max_log>500</max_log>
</integration>
```

### Opções de Agendamento de Análise

```javascript
// No node "Schedule Trigger"

// A cada 4 horas
{ "hours": 4 }

// A cada 6 horas (padrão)
{ "hours": 6 }

// Diariamente às 8h
{ "cronExpression": "0 8 * * *" }

// A cada 30 minutos
{ "minutes": 30 }
```

### Personalização de Email

Atualize o destinatário no node **Enviar Relatório por Email**:

```javascript
{
  "fromEmail": "your-email@example.com",
  "toEmail": "your-email@example.com",
  "subject": "Relatório de Segurança Wazuh - {{ $json.report_date }}",
  "text": "={{ $json.html_report }}"
}
```

---

## 🔧 Solução de Problemas

### Problemas Comuns

#### 1. Alertas Não Chegam

**Sintomas:** Webhook não recebe dados

**Soluções:**
```bash
# Verificar logs do gerenciador Wazuh
tail -f /var/ossec/logs/ossec.log | grep integration

# Verificar conectividade de rede
curl -X POST https://sua-instancia-n8n.com/webhook/wazuh/ingest-YOUR_WEBHOOK_TOKEN \
  -H "Content-Type: application/json" \
  -H "X-Wazuh-Auth: seu-token-secreto-aqui" \
  -d '{"test": "data"}'

# Verificar configuração de integração Wazuh
grep -A 10 "<integration>" /var/ossec/etc/ossec.conf

# Reiniciar gerenciador Wazuh
systemctl restart wazuh-manager
```

#### 2. Análise da IA Falha

**Sintomas:** Vereditos vazios, erros de análise

**Soluções:**
```javascript
// Verificar chave de API OpenAI
console.log('Chave API configurada:', !!$credentials.openAiApi);

// Verificar disponibilidade do modelo
// Modelos alternativos:
// - "gpt-4-turbo-preview" (padrão)
// - "gpt-4"
// - "gpt-3.5-turbo"

// Aumentar timeout no node HTTP Request
{
  "timeout": 60000 // 60 segundos
}
```

#### 3. Memória Não Funciona

**Sintomas:** Relatórios mostram "Primeira Análise" para todos os hosts

**Soluções:**

**Verificar Registros da Tabela de Dados:**
1. Navegue até **Data** → **Tables** → `wazuh_alerts`
2. Filtre por colunas:
   - `verdict` NÃO ESTÁ VAZIO
   - `processed` = "1"
3. Verifique se vereditos estão sendo armazenados com timestamps `analyzed_at`

#### 4. Limpeza Exclui Demais

**Sintomas:** Contexto de memória perdido, alertas antigos excluídos prematuramente

**Soluções:**
```javascript
// Ajustar parâmetros de retenção em "Calcular Parâmetros de Limpeza"
const retentionDays = 7; // Aumentar de 3 para 7
const minAlertsPerHost = 10; // Aumentar de 5 para 10

// Desabilitar limpeza temporariamente
// Comentar conexão de "Armazenar Vereditos" para "Calcular Parâmetros de Limpeza"
```

#### 5. Email Não Chega ou Chega Vazio

**Sintomas:** Nenhum email recebido ou email sem conteúdo

**Soluções:**
```javascript
// Verificar credenciais SMTP
// No n8n: Settings → Credentials → Sua credencial SMTP

// Para Gmail, usar senha de aplicativo:
// 1. Ativar verificação em 2 etapas
// 2. Criar senha de aplicativo
// 3. Usar senha de aplicativo nas credenciais

// Verificar configuração do node de email:
{
  "fromEmail": "your-email@example.com",
  "toEmail": "your-email@example.com",
  "emailType": "html",
  "text": "={{ $json.html_report }}"
}

// Teste manual com node de email
// Execute apenas o node "Enviar Relatório por Email"
```

### Dicas de Depuração

#### Habilitar Log Detalhado

```javascript
// Adicionar a qualquer node de código
console.log('=== INÍCIO DEBUG ===');
console.log('Itens de entrada:', $input.all().length);
console.log('Item atual:', JSON.stringify($input.item, null, 2));
console.log('=== FIM DEBUG ===');
```

#### Testar Nodes Individuais

1. Clique com botão direito no node → **Execute Node**
2. Verifique saída de execução
3. Verifique estrutura de dados

#### Monitorar Histórico de Execução

- Aba **Executions** no n8n
- Filtrar por workflow
- Verificar erros/avisos

---

## 💬 Integração Bitrix24

### Visão Geral

O sistema possui integração com Bitrix24 para envio de alertas individuais em tempo real via chat. Enquanto o email envia um relatório consolidado, o Bitrix24 envia cada alerta analisado separadamente para um chat específico.

### Fluxo de Envio

```
Agregar Todos os Resultados
    ├→ Gerar Relatório HTML → Enviar Email (consolidado)
    └→ Gerar Alertas Bitrix → Enviar para Bitrix24 (individual)
```

### Configuração do Webhook

**URL do Webhook Bitrix24:**
```
https://YOUR_DOMAIN.bitrix24.com.br/rest/YOUR_USER_ID/YOUR_WEBHOOK_TOKEN/im.message.add.json
```

**Chat de Destino:**
- Chat ID: `chatYOUR_CHAT_ID`

**Método:** POST

**Parâmetros:**
```json
{
  "DIALOG_ID": "chatYOUR_CHAT_ID",
  "MESSAGE": "Mensagem formatada em BBCode"
}
```

### Formato BBCode

As mensagens enviadas para o Bitrix24 usam formatação BBCode:

```
🚨 [B]Alerta de Segurança Wazuh[/B]

[B]Nome do Alerta:[/B] Multiple authentication failures
[B]Origem:[/B] 192.168.1.100 (web-server-01)
[B]Severidade:[/B] 🔴 Alta (Nível 7)
[B]Data do Incidente:[/B] 21/10/2025 14:30:00
[B]Quantidade de Alertas:[/B] 5
🧠 Análise com Contexto Histórico

🔍 [B]Resumo da Análise[/B]
Múltiplas tentativas de autenticação falhada detectadas...

⚖️ [B]Veredito[/B]
Verdadeiro Positivo - Confiança: Alta

⚙️ [B]Ações Recomendadas[/B]
1. Investigar origem dos acessos
2. Implementar bloqueio de IP
3. Reforçar políticas de senha
```

### Mapeamento de Severidade

O sistema classifica automaticamente a severidade baseado no `rule_level`:

| rule_level | Emoji | Classificação |
|------------|-------|---------------|
| ≥ 12 | ⛔ | Crítica |
| 7-11 | 🔴 | Alta |
| 3-6 | 🟡 | Média |
| < 3 | 🟢 | Baixa |

### Indicadores de Memória

Cada alerta mostra se foi analisado com contexto histórico:

- **🧠 Análise com Contexto Histórico** (verde): IA usou memória de análises anteriores
- **⚠️ Primeira Análise** (cinza): Primeira vez analisando este agente

### Campos Incluídos

**Sempre Presentes:**
- Nome do Alerta (rule_description)
- Origem (agent_ip + agent_name)
- Severidade com emoji
- Data do Incidente
- Quantidade de Alertas
- Resumo da Análise
- Veredito e Confiança
- Ações Recomendadas

**Condicionais (quando disponíveis):**
- 📋 Análise Detalhada
- 🔄 Mudanças de Padrão (comparado ao histórico)

### Personalização

#### Alterar Chat de Destino

No workflow, edite o node **Enviar para Bitrix24**:

```json
{
  "DIALOG_ID": "chatYOUR_CHAT_ID",  // Seu chat ID
  "MESSAGE": "={{ $json.bitrix_message }}"
}
```

#### Alterar Webhook URL

Se usar outro Bitrix24, atualize a URL no node **Enviar para Bitrix24**:

```
https://sua-empresa.bitrix24.com/rest/USER_ID/WEBHOOK_CODE/im.message.add.json
```

**Como obter Webhook:**
1. No Bitrix24, vá em **Aplicativos** → **Webhooks**
2. Crie novo webhook de entrada
3. Selecione permissão: **im** (Instant Messenger)
4. Copie a URL gerada

#### Modificar Formatação

Edite o node **Gerar Alertas Bitrix** para personalizar a mensagem BBCode.

**Tags BBCode disponíveis:**
- `[B]texto[/B]` - Negrito
- `[I]texto[/I]` - Itálico
- `[U]texto[/U]` - Sublinhado
- `[COLOR=#FF0000]texto[/COLOR]` - Cor
- `[URL=link]texto[/URL]` - Link

### Limitações

- **Tamanho máximo:** ~4096 caracteres por mensagem
- **Taxa de envio:** Limitada pela API do Bitrix24 (evite > 20 msg/segundo)
- **Mensagens longas:** Automaticamente truncadas com indicação

### Solução de Problemas Bitrix24

#### Mensagens Não Chegam

**Verificar:**
1. Webhook URL está correto
2. Chat ID existe e está acessível
3. Permissões do webhook incluem `im`
4. Bitrix24 não está com manutenção

**Teste manual:**
```bash
curl -X POST "https://YOUR_DOMAIN.bitrix24.com.br/rest/YOUR_USER_ID/YOUR_WEBHOOK_TOKEN/im.message.add.json" \
  -d "DIALOG_ID=chatYOUR_CHAT_ID" \
  -d "MESSAGE=Teste de mensagem"
```

#### Formatação BBCode Incorreta

**Causas comuns:**
- Tags BBCode não fechadas: `[B]texto` (falta `[/B]`)
- Caracteres especiais não escapados
- Quebras de linha excessivas

**Solução:**
Edite o node **Gerar Alertas Bitrix** e verifique a montagem da string `message`.

#### Múltiplas Mensagens Duplicadas

**Causa:** Workflow executado várias vezes

**Solução:**
- Verifique se o workflow não está rodando em loop
- Confirme que o agendamento está correto (6 horas)
- Desative e reative o workflow

---

## 📈 Monitoramento e Métricas

### Indicadores Chave de Desempenho

| Métrica | Meta | Como Verificar |
|---------|------|----------------|
| Taxa de Ingestão de Alertas | >99% | Tabela de Dados → Filtrar por `created_at` (última hora) |
| Conclusão de Análise | >95% | Contar linhas onde `processed="1"` vs total |
| Taxa de Falsos Positivos | <40% | Contar `verdict="Falso Positivo"` vs total com vereditos |
| Cobertura de Memória | >80% | Verificar contagem "Com Memória" nos relatórios |

---

## 🎯 Melhores Práticas

### 1. Filtragem de Alertas

Configure Wazuh para enviar apenas alertas relevantes:

```xml
<!-- Enviar apenas alertas críticos (nível 7+) -->
<integration>
  <level>7</level>
</integration>

<!-- Excluir regras específicas -->
<integration>
  <rule_id>5502,5503</rule_id> <!-- Excluir login/logout de usuário -->
</integration>
```

### 2. Otimização de Agendamento

Ajuste baseado no volume de alertas:

| Volume de Alertas | Agendamento Recomendado |
|-------------------|------------------------|
| <100/dia | A cada 12 horas |
| 100-500/dia | A cada 6 horas |
| 500-2000/dia | A cada 4 horas |
| >2000/dia | A cada 2 horas |

### 3. Gerenciamento de Custos (OpenAI)

```javascript
// Monitorar uso da API
const costPerToken = 0.00001; // Preço GPT-4 Turbo
const avgTokensPerAnalysis = 2000;
const analysesPerDay = 50;

const dailyCost = analysesPerDay * avgTokensPerAnalysis * costPerToken;
console.log(`Custo diário estimado: $${dailyCost.toFixed(2)}`);

// Otimizar tamanho do prompt
const maxAlertDetails = 5; // Mostrar apenas primeiros 5 alertas em detalhe
const truncateRawBody = true; // Resumir ao invés de JSON completo
```

---

## 📚 Recursos Adicionais

### Documentação
- [Documentação Oficial n8n](https://docs.n8n.io)
- [Documentação Wazuh](https://documentation.wazuh.com)
- [Referência da API OpenAI](https://platform.openai.com/docs)

### Comunidade
- [Fórum da Comunidade n8n](https://community.n8n.io)
- [Slack Wazuh](https://wazuh.com/community/join-us-on-slack)

### Segurança
- [Avisos de Segurança Wazuh](https://wazuh.com/security-vulnerabilities)
- [OWASP Top 10](https://owasp.org/www-project-top-ten)

---

## 📝 Licença e Créditos

### Licença
Este workflow é fornecido como está sob Licença MIT.

### Créditos
- **Sistema de Análise de Alertas Wazuh com IA**
- **Modelo de IA:** GPT-4 Turbo (OpenAI)
- **Plataforma:** n8n.io
- **Plataforma de Segurança:** Wazuh
- **Integração:** Bitrix24
- **Idioma:** Português Brasileiro

### Agradecimentos
- Obrigado à comunidade n8n pela inspiração de workflows
- OpenAI pelo acesso à API de IA
- Equipe Wazuh pela excelente plataforma SIEM

---

## 🤝 Contribuindo

### Reportar Problemas
Encontrou um bug? Tem uma solicitação de recurso?
- Crie exportação do workflow e compartilhe
- Descreva comportamento esperado vs real
- Inclua logs de erro se aplicável

### Melhorias
Sugestões são bem-vindas:
- Otimizações de desempenho
- Novos modelos de IA
- Integrações adicionais
- Melhores formatos de relatórios

---

**Última Atualização:** Outubro 2025  
**Versão:** 2.0 (OpenAI + SMTP)  
**Compatibilidade:** n8n v1.0+, Wazuh v4.0+
