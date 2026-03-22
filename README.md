# 💅 Shay Finanças — Beauty Finance

## Estrutura do Projeto

```
shay-financas/
├── beauty-finance.html    ← App principal (abrir no browser)
├── db/
│   └── shay-financas-db.xlsx  ← Base de dados Excel
└── README.md
```

## Como Usar

1. **Abrir a app**: Abre `beauty-finance.html` no teu browser
2. **Adicionar dados manualmente**: Usa os botões "+ Nova Receita" e "+ Nova Despesa"
3. **Importar Excel**: Vai a "Importar Excel" e arrasta o ficheiro `db/shay-financas-db.xlsx`
4. **Exportar relatório**: Em "Relatórios" clica "Exportar Relatório Excel"

## Base de Dados Excel (db/shay-financas-db.xlsx)

O ficheiro Excel contém 5 folhas:

| Folha | Descrição |
|-------|-----------|
| **Receitas** | Todos os serviços realizados (com comissão calculada) |
| **Despesas** | Pessoais, profissionais, marketing e produtos |
| **Resumo Mensal** | Vista anual com fórmulas automáticas |
| **Serviços** | Tabela de preços de referência |
| **Configurações** | Parâmetros (comissão, moeda, dados pessoais) |

## Lógica de Comissão

- **Salão (50%)**: Serviços feitos no espaço do salão → 50% vai para o salão
- **Sala Extra (100%)**: Gabinete próprio → ficas com 100% da receita

## Tecnologias

- HTML5 + CSS3 + JavaScript
- Bootstrap 5 (layout responsivo)
- GSAP (animações)
- Chart.js (gráficos)
- SheetJS (import/export Excel)
- localStorage (persistência no browser)
