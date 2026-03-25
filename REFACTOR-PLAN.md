# S·Finance — Plano de Refactoring: Function Calling

## Resumo
Substituir o router AI actual (texto → parse JSON → execute) por Gemini Function Calling nativo.

## Fases

### FASE 1: Preparar aiCall para tools (10 min)
**Ficheiro:** beauty-finance.html (linha ~2093)

Adicionar suporte para `tools` e `tool_config` no body do Gemini:
```javascript
// ANTES:
const body = {contents, generationConfig: {...}};
if(systemText) body.systemInstruction = {...};

// DEPOIS:
const body = {contents, generationConfig: {...}};
if(systemText) body.systemInstruction = {...};
if(config.tools) body.tools = config.tools;
if(config.toolConfig) body.tool_config = config.toolConfig;
```

Alterar o return para devolver o objecto completo (não só texto):
```javascript
// ANTES:
return data.candidates?.[0]?.content?.parts?.[0]?.text || '';

// DEPOIS:
if(config.rawResponse) return data; // para function calling
return data.candidates?.[0]?.content?.parts?.[0]?.text || '';
```

### FASE 2: Declarar as function tools (15 min)
**Novo bloco:** `const AI_TOOLS = [{ functionDeclarations: [...] }]`

8 funções:
1. `consultar_registos` — SELECT com filtros
2. `alterar_registo` — UPDATE campo WHERE id
3. `apagar_registo` — DELETE WHERE id  
4. `criar_evento` — INSERT evento
5. `criar_nota` — INSERT nota
6. `registar_receita` — INSERT receita (com confirmação)
7. `registar_despesa` — INSERT despesa (com confirmação)
8. `executar_sql` — SQL livre (com confirmação)
9. `responder` — resposta texto puro (chat/ajuda)

### FASE 3: Criar executors para cada função (15 min)
**Novo bloco:** `const TOOL_EXECUTORS = { consultar_registos, alterar_registo, ... }`

Cada executor:
- Recebe args do Gemini
- Valida segurança
- Executa SQL via run()/q1()
- Retorna resultado estruturado
- Actualiza lastContext

### FASE 4: Reescrever routeAndExecute (20 min)
**Substituir:** linhas ~2583-2700 (o router inteiro)

Novo fluxo:
```
1. Montar contents (history + mensagem)
2. Chamar aiCall com tools + rawResponse:true
3. Se response tem functionCall:
   a. Executar via TOOL_EXECUTORS
   b. Montar functionResponse
   c. 2ª chamada ao Gemini com resultado
   d. Mostrar resposta texto do Gemini
4. Se response tem texto:
   a. Mostrar directamente (é chat)
```

### FASE 5: Remover código morto (10 min)
**Remover:**
- INTENT_SCHEMA, intentToDecision()
- ROUTER_PROMPT  
- getIntentSystem()
- buildLastContext()
- queryThinkLoop(), formatFinalAnswer()
- processAiReply()
- Simplificar getDataContext() (menos verbose)
- Simplificar getSchemaForAI() (só para system prompt)

### FASE 6: Adaptar sendAiMsg (5 min)
**Simplificar:** já não precisa de ctx separado
```javascript
// ANTES:
const ctx = getDataContext();
await routeAndExecute(msg, hasImage, ctx);

// DEPOIS:
await routeAndExecute(msg, hasImage);
```

### FASE 7: Adaptar confirmAiAction para receitas/despesas (5 min)
Receitas e despesas continuam com confirmação via card.
O Gemini chama `registar_receita` → mostra card → user confirma → executa.

### FASE 8: Fallback Anthropic (10 min)
O Anthropic Claude não suporta function calling via REST da mesma forma.
Manter o ROUTER_PROMPT simplificado como fallback para Anthropic.

## Tempo total estimado: ~90 min

## O que NÃO muda
- UI do chat (HTML + CSS)
- Sistema de chats (create, load, delete, list)
- Preview cards (renderCard)
- GitHub deploy (handleAppModification)
- Toda a lógica da app fora da AI (dashboard, receitas, despesas, etc)
- aiCall base (só adicionar tools support)

## Riscos
1. O modelo Gemini 2.5 Flash free tier suporta function calling? → SIM (desde 1.5)
2. A REST API suporta tools? → SIM (documentação oficial)
3. Limites de funções? → Até 128 function declarations
4. Imagens + function calling? → NÃO ao mesmo tempo (limitação). Fallback para texto.
