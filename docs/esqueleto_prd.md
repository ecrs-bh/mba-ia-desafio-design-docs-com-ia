### PRD: [produto] [feature]

Versão: [versao]
Data: [data]
Responsável: [responsavel_prd]

---

### Resumo

[contexto.resumo]

---

### Contexto e problema

Público-alvo
- [público alvo 1]
- [público alvo 2]

Cenários de uso chave
- [cenário 1]
- [cenário 2]

Onde essa feature será implantada
- [contexto_implantacao.descricao]

Problemas priorizados
- [problema 1 com impacto e prioridade]
- [problema 2 com impacto e prioridade]

---

### Objetivos e métricas

| Objetivo                                                               | Métrica                                                         | Meta                      |
| ---------------------------------------------------------------------- | --------------------------------------------------------------- | ------------------------- |
| [objetivo 1]                                                           | [métrica 1]                                                     | [meta 1]                  |
| [objetivo 2]                                                           | [métrica 2]                                                     | [meta 2]                  |

---

### Escopo

Incluso
- [item incluso 1]
- [item incluso 2]

Fora de escopo
- [item fora 1]
- [item fora 2]

---

### Requisitos funcionais

#### [id] [nome do requisito]
[descricao do requisito]

**Fluxo principal**
- [passo 1]
- [passo 2]

**Fluxos alternativos e exceções**
- [variação / exceção 1]
- [variação / exceção 2]

**Erros previstos**
- [erro previsto 1]
- [erro previsto 2]

**Prioridade:** [alta|media|baixa]

---

#### [id] [nome do requisito 2]
[descricao do requisito 2]

**Fluxo principal**
- [passo 1]
- [passo 2]

**Fluxos alternativos e exceções**
- [variação / exceção]

**Erros previstos**
- [erro previsto]

**Prioridade:** [alta|media|baixa]

---

### Requisitos não funcionais

Performance
- [ex: p95 menor que 150 ms]

Disponibilidade
- [ex: 99.9 por cento de uptime mensal em produção]

Segurança e autorização
- [ex: autenticação obrigatória e auditoria de alterações sensíveis]

Observabilidade
- [ex: logs estruturados, métricas de erro por endpoint, tracing distribuído ponta a ponta]

Confiabilidade e integridade de dados
- [ex: atualização de estoque deve ser transacional]

Compatibilidade e portabilidade
- [ex: APIs REST JSON versionado /v1, empacotado em container OCI]

Compliance
- [ex: trilha de auditoria de preço e estoque disponível para reconciliação]

Acessibilidade no frontend consumidor
- [ex: resposta da API traz texto alternativo de imagem e rótulos necessários para acessibilidade]

---

### Arquitetura e abordagem

Abordagem
- [descrição da abordagem geral. ex: microsserviço dedicado responsável por produto, SKU, estoque e preço]

Componentes
- [componente 1. ex: API backend em Go]
- [componente 2. ex: banco PostgreSQL como fonte de verdade]
- [componente 3]

Integrações
- [integração 1. ex: checkout consome snapshot de preço e estoque no carrinho]
- [integração 2]

### Decisões e trade-offs

#### Decisão: [decisão 1]
- **Justificativa:** [por que essa decisão foi tomada]
- **Trade-off:** [custo ou limitação associada]

#### Decisão: [decisão 2]
- **Justificativa:** [por que essa decisão foi tomada]
- **Trade-off:** [custo ou limitação associada]

---

### Dependências

#### [tipo da dependência]: [título]
[descrição da dependência, incluindo quem precisa entregar o quê e por quê]

#### [tipo da dependência]: [título 2]
[descrição da dependência 2]

---

### Riscos e mitigação

#### [risco 1 resumido em uma frase]
- **Probabilidade:** [baixa|media|alta]
- **Impacto:** [impacto esperado]
- **Mitigação:**
  - [ação de mitigação 1]
  - [ação de mitigação 2]
- **Plano de contingência:** [plano B se der errado]

#### [risco 2 resumido em uma frase]
- **Probabilidade:** [baixa|media|alta]
- **Impacto:** [impacto esperado]
- **Mitigação:**
  - [ação de mitigação 1]
- **Plano de contingência:** [plano B se der errado]

---

### Critérios de aceitação
Checklist objetivo que define se a feature está pronta.

- [critério 1]
- [critério 2]
- [critério 3]

---

### Testes e validação

Tipos de teste obrigatórios
- [tipo de teste 1. ex: testes unitários para regras críticas]
- [tipo de teste 2. ex: testes de integração para fluxo principal]
- [tipo de teste 3. ex: teste de segurança de permissão de alteração de preço]

Estratégia de validação
- [ex: TDD para lógica crítica de estoque e preço, QA manual guiado por roteiro, validação exploratória navegando na vitrine com dados reais]

