# 🚀 Guia de Início Rápido

## Configuração em 10 Passos

### 1. Criar Tabela de Dados no n8n
```
n8n → Data → Tables → Add Table
Nome: wazuh_alerts
```

### 2. Configurar Credenciais OpenAI
```
n8n → Settings → Credentials → Add Credential → OpenAI API
Cole sua chave API do OpenAI
```

### 3. Configurar Credenciais SMTP
```
n8n → Settings → Credentials → Add Credential → SMTP
Host: smtp.gmail.com (ou seu servidor)
Port: 587
User: seu-email@gmail.com
Password: senha-de-aplicativo
```

### 4. Configurar Autenticação Header (Segurança)
```
n8n → Settings → Credentials → Add Credential → Header Auth
Name: Header Wazuh
Header Name: X-Wazuh-Auth
Value: gerar-token-aleatorio-seguro-aqui
```

### 5. Importar Workflow 1 (Coleta)
```
n8n → Workflows → Import from File
Selecione: workflow1-clean.json
```

### 6. Configurar Workflow 1
```
Abra o workflow importado
Clique em "Wazuh Webhook"
- Selecione credencial: Header Wazuh
- Copie a URL do webhook

Clique em "Inserir na Tabela de Dados"
- Selecione: wazuh_alerts
```

### 7. Importar Workflow 2 (Análise)
```
n8n → Workflows → Import from File
Selecione: workflow2-clean.json
```

### 8. Configurar Workflow 2
```
Abra o workflow importado

Node "Análise de IA":
- Selecione credencial OpenAI

Node "Enviar Relatório por Email":
- fromEmail: seu-email@example.com
- toEmail: destinatario@example.com
- Selecione credencial SMTP

Node "Enviar para Bitrix24" (opcional):
- URL: seu webhook Bitrix24
- DIALOG_ID: seu chat ID
```

### 9. Configurar Wazuh
```bash
# Editar configuração
sudo nano /var/ossec/etc/ossec.conf

# Adicionar integração
<integration>
  <name>custom-webhook</name>
  <hook_url>SUA_URL_DO_WEBHOOK_AQUI</hook_url>
  <level>3</level>
  <alert_format>json</alert_format>
</integration>

# Reiniciar
sudo systemctl restart wazuh-manager
```

### 10. Ativar e Testar
```
n8n → Ative ambos os workflows
Trigger um alerta no Wazuh
Verifique a Tabela de Dados
Aguarde o próximo agendamento (6h) ou execute manualmente
```

## 📋 Checklist de Configuração

- [ ] Tabela de dados `wazuh_alerts` criada
- [ ] Credencial OpenAI configurada
- [ ] Credencial SMTP configurada
- [ ] Credencial Header Auth configurada (recomendado)
- [ ] Workflow 1 importado e configurado
- [ ] Workflow 2 importado e configurado
- [ ] URL do webhook copiada
- [ ] Wazuh configurado com URL do webhook
- [ ] Wazuh reiniciado
- [ ] Workflows ativados
- [ ] Teste de alerta realizado

## 🔧 Comandos Úteis

### Testar Webhook
```bash
curl -X POST "SUA_URL_DO_WEBHOOK" \
  -H "Content-Type: application/json" \
  -H "X-Wazuh-Auth: SEU_TOKEN_AQUI" \
  -d '{
    "agent": {
      "name": "test-server",
      "ip": "192.168.1.100",
      "id": "001"
    },
    "rule": {
      "id": "5710",
      "level": 7,
      "description": "Teste de alerta",
      "groups": ["authentication"]
    },
    "message": "Teste manual do webhook",
    "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
    "location": "/var/log/test.log"
  }'
```

### Verificar Logs Wazuh
```bash
# Ver integração
tail -f /var/ossec/logs/ossec.log | grep integration

# Ver todos os alertas
tail -f /var/ossec/logs/alerts/alerts.json
```

### Verificar Tabela de Dados
```
n8n → Data → Tables → wazuh_alerts
Ver registros recentes
Filtrar por: processed = "0" (não processados)
```

## ⚠️ Troubleshooting Rápido

### Webhook não recebe dados
```bash
# Verificar se Wazuh está enviando
grep integration /var/ossec/logs/ossec.log

# Testar conectividade
curl -I SUA_URL_DO_WEBHOOK
```

### Análise da IA não funciona
```
Verificar:
1. Credencial OpenAI está correta?
2. Tem saldo na conta OpenAI?
3. Modelo GPT-4 está disponível?
```

### Email não chega
```
Verificar:
1. Credenciais SMTP estão corretas?
2. Para Gmail: Usando senha de aplicativo?
3. Porta 587 está aberta?
4. Email não está no spam?
```

### Alertas não são processados
```
Verificar:
1. Workflow 2 está ativo?
2. Agendamento está configurado?
3. Há alertas com processed="0"?
4. Execute o workflow manualmente
```

## 🎯 Próximos Passos

1. **Ajustar Filtros Wazuh**
   - Definir nível mínimo de severidade
   - Excluir regras irrelevantes
   - Adicionar regras específicas

2. **Personalizar Análise IA**
   - Ajustar temperatura do modelo
   - Modificar prompt para seu contexto
   - Adicionar instruções específicas

3. **Configurar Alertas**
   - Adicionar mais destinatários de email
   - Configurar Bitrix24/Slack/Discord
   - Criar webhooks para outras ferramentas

4. **Otimizar Limpeza**
   - Ajustar dias de retenção
   - Modificar quantidade de alertas por host
   - Criar regras de limpeza customizadas

5. **Monitorar Performance**
   - Verificar execuções no n8n
   - Monitorar custos da API OpenAI
   - Revisar taxa de falsos positivos

## 📚 Recursos Adicionais

- [README.md](./README.md) - Documentação completa
- [CHANGELOG.md](./CHANGELOG.md) - Histórico de alterações
- [Documentação n8n](https://docs.n8n.io)
- [Documentação Wazuh](https://documentation.wazuh.com)
- [Documentação OpenAI](https://platform.openai.com/docs)

## 💡 Dicas

- 🔒 **Segurança:** Sempre use autenticação Header para webhooks
- 💰 **Custos:** Monitore uso da API OpenAI regularmente
- 📊 **Relatórios:** Revise relatórios semanalmente para ajustar sistema
- 🧹 **Limpeza:** Ajuste retenção baseado no seu volume de alertas
- 🎯 **Precisão:** Feedback contínuo melhora a qualidade da IA

---

**Tempo estimado de configuração:** 30-45 minutos  
**Dificuldade:** Intermediário  
**Última atualização:** Outubro 2025

