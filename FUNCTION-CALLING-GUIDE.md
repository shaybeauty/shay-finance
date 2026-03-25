# Guia: Gemini Function Calling para S·Finance

## O Problema Actual
Estamos a pedir ao Gemini para fazer tudo numa só chamada: classificar intent, gerar SQL, manter contexto, corrigir erros. Isto falha constantemente.

## A Solução: Function Calling (Padrão oficial Google)
Em vez de pedir texto/JSON ao Gemini, declaramos **funções** que ele pode chamar. O Gemini decide qual chamar e com que parâmetros. O nosso código executa.

## Como funciona (REST API)

### 1. Declarar funções (tools)
```javascript
const tools = [{
  functionDeclarations: [
    {
      name: "consultar_registos",
      description: "Consulta eventos, notas, receitas ou despesas na base de dados",
      parameters: {
        type: "object",
        properties: {
          tabela: { type: "string", enum: ["eventos","notas","receitas","despesas"], description: "Tabela a consultar" },
          filtro: { type: "string", description: "Filtro de pesquisa: titulo, data, etc" }
        },
        required: ["tabela"]
      }
    },
    {
      name: "alterar_registo",
      description: "Altera um campo de um registo existente (evento, nota, receita, despesa)",
      parameters: {
        type: "object",
        properties: {
          tabela: { type: "string", enum: ["eventos","notas","receitas","despesas"] },
          id: { type: "string", description: "ID do registo a alterar" },
          campo: { type: "string", description: "Nome do campo: time_start, date_start, title, amount, etc" },
          valor: { type: "string", description: "Novo valor" }
        },
        required: ["tabela","id","campo","valor"]
      }
    },
    {
      name: "criar_evento",
      description: "Cria um novo evento na agenda",
      parameters: {
        type: "object",
        properties: {
          title: { type: "string" },
          date_start: { type: "string", description: "Data YYYY-MM-DD" },
          time_start: { type: "string", description: "Hora HH:MM" },
          time_end: { type: "string", description: "Hora fim HH:MM" },
          tipo: { type: "string", enum: ["pessoal","lembrete","habito","bloqueio"] }
        },
        required: ["title","date_start"]
      }
    },
    {
      name: "criar_nota",
      description: "Cria uma nova nota",
      parameters: { ... }
    },
    {
      name: "registar_receita",
      description: "Regista uma receita/serviço",
      parameters: { ... }
    },
    {
      name: "registar_despesa",
      description: "Regista uma despesa",
      parameters: { ... }
    },
    {
      name: "apagar_registo",
      description: "Apaga um registo da base de dados",
      parameters: {
        type: "object",
        properties: {
          tabela: { type: "string", enum: ["eventos","notas","receitas","despesas"] },
          id: { type: "string", description: "ID do registo a apagar" }
        },
        required: ["tabela","id"]
      }
    },
    {
      name: "executar_sql",
      description: "Executa SQL para consultas complexas ou alterações de categorias",
      parameters: {
        type: "object",
        properties: {
          sql: { type: "string", description: "Query SQL" },
          descricao: { type: "string" }
        },
        required: ["sql"]
      }
    }
  ]
}];
```

### 2. Enviar ao Gemini via REST API
```javascript
const body = {
  contents: [
    // histórico de conversa
    ...history,
    // mensagem actual
    { role: "user", parts: [{ text: "altera a hora desse evento para as 19:00" }] }
  ],
  tools: tools,
  systemInstruction: { parts: [{ text: systemPrompt }] },
  generationConfig: { temperature: 0.1 }
};

const resp = await fetch(`${API_URL}?key=${API_KEY}`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(body)
});
const data = await resp.json();
```

### 3. O Gemini responde com functionCall (NÃO texto!)
```json
{
  "candidates": [{
    "content": {
      "parts": [{
        "functionCall": {
          "name": "alterar_registo",
          "args": {
            "tabela": "eventos",
            "id": "lz4abc1x",
            "campo": "time_start",
            "valor": "19:00"
          }
        }
      }]
    }
  }]
}
```

### 4. O nosso código executa a função
```javascript
const part = data.candidates[0].content.parts[0];

if (part.functionCall) {
  const { name, args } = part.functionCall;
  let result;
  
  switch (name) {
    case 'alterar_registo':
      run(`UPDATE ${args.tabela} SET ${args.campo}=? WHERE id=?`, [args.valor, args.id]);
      result = { sucesso: true, mensagem: `${args.campo} alterado para ${args.valor}` };
      break;
    case 'consultar_registos':
      result = execQuery(`SELECT * FROM ${args.tabela} WHERE title LIKE '%${args.filtro}%'`);
      break;
    case 'criar_evento':
      const id = uid();
      run(`INSERT INTO eventos ...`, [id, args.title, args.date_start, ...]);
      result = { sucesso: true, id: id, mensagem: `Evento ${args.title} criado` };
      break;
    // ... outros casos
  }
  
  // PASSO 5: Devolver resultado ao Gemini para resposta final
}
```

### 5. Devolver resultado ao Gemini (2ª chamada)
```javascript
// Adicionar ao histórico:
// - A functionCall do Gemini
// - O nosso functionResponse

const followUp = {
  contents: [
    ...history,
    { role: "user", parts: [{ text: mensagem }] },
    // O que o Gemini pediu
    { role: "model", parts: [{ functionCall: { name, args } }] },
    // O resultado da nossa execução
    { role: "user", parts: [{ functionResponse: { name, response: result } }] }
  ],
  tools: tools,
  systemInstruction: { parts: [{ text: systemPrompt }] }
};

const resp2 = await fetch(`${API_URL}?key=${API_KEY}`, { ... });
// Agora o Gemini responde com texto natural:
// "A hora do evento Cabeleireira foi alterada para as 19:00."
```

## Vantagens vs o que temos agora
| Actual | Function Calling |
|---|---|
| Gemini gera SQL (erra frequentemente) | Gemini escolhe função + parâmetros |
| Parse de JSON do texto (falha) | functionCall é objecto nativo |
| Contexto perdido entre mensagens | history mantém contexto automaticamente |
| 1 chamada tenta fazer tudo | 2 chamadas com papéis claros |
| Structured output complexo | Funções simples com parâmetros simples |

## Fluxo completo para "altera a hora desse evento"

```
User: "que eventos tenho?"
  → Gemini chama: consultar_registos(tabela="eventos")
  → Código executa: SELECT * FROM eventos
  → Resultado: [{id:"abc", title:"Cabeleireira", time_start:"09:00"}]
  → Gemini recebe resultado → responde: "Tens 1 evento: Cabeleireira dia 18/04 às 09:00"
  → lastContext = {id:"abc", tabela:"eventos"}

User: "altera a hora desse para as 19:00"  
  → Gemini vê history + lastContext
  → Gemini chama: alterar_registo(tabela="eventos", id="abc", campo="time_start", valor="19:00")
  → Código executa: UPDATE eventos SET time_start='19:00' WHERE id='abc'
  → Resultado: {sucesso: true}
  → Gemini responde: "Hora da Cabeleireira alterada para as 19:00 ✅"
```

## Referências
- Documentação oficial: https://ai.google.dev/gemini-api/docs/function-calling
- REST API cookbook: https://github.com/google-gemini/cookbook/blob/main/quickstarts/rest/Function_calling_REST.ipynb
- SQL Talk App (Google): https://github.com/GoogleCloudPlatform/generative-ai/tree/main/gemini/function-calling/sql-talk-app
- TypeScript + SQLite example: https://carboleda.medium.com/from-zero-to-hero-in-llm-function-calling-with-google-gemini-and-typescript-efb916aee60f
