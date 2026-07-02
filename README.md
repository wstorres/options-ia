# PROJETO: OptionsIA — Agente de Recomendação para Compra de Opções (Calls/Puts OTM)

> Documento de especificação técnica para desenvolvimento via Google Antigravity (multiagente).
> Objetivo: servir de contexto de projeto para os agentes lerem e executarem o build.

---

## 1. Visão Geral

Aplicação web para apoiar decisões de **compra de opções (Call ou Put) fora do dinheiro (OTM)**,
com foco no mercado brasileiro (B3). O sistema **não executa ordens** — é uma ferramenta de
**análise e recomendação**. Quem decide e executa é sempre o operador humano.

O núcleo do produto é um **Agente de IA especialista em opções**, que analisa:
- Volatilidade (implícita e histórica)
- Gregas (Delta, Gamma, Theta, Vega, e opcionalmente Rho)
- Liquidez do contrato (volume, número de negócios, spread bid/ask)
- Tendência do ativo-objeto (indicadores técnicos)
- Distância do strike em relação ao preço atual (moneyness)
- Tempo até o vencimento (theta decay)

E devolve recomendações estruturadas: qual ativo, qual strike, qual vencimento, tipo (Call/Put),
tese de tendência, tamanho de posição sugerido (dentro da regra de 10 operações), e nível de risco.

**Disclaimer obrigatório no produto:** a ferramenta é um apoio analítico, não uma recomendação de
investimento personalizada nem garantia de resultado. Operações com opções envolvem risco de perda
total do capital alocado na opção.

---

## 2. Estratégia de Investimento a Modelar (regras de negócio)

Este é o "cérebro" que o Agente de IA precisa internalizar como regra fixa:

1. **Somente compra** de Call ou Put (nunca venda/lançamento coberto ou descoberto nesta v1).
2. **OTM (Out of the Money)** — preferencialmente opções OTM ou bem OTM ("pozinho"), priorizando
   prêmio baixo e potencial de valorização assimétrica (payoff convexo).
3. **Tese central:** comprar opções baratas na expectativa de que o ativo-objeto se mova em direção
   ao strike (ou o ultrapasse), valorizando a opção de forma desproporcional ao custo pago.
4. **Gestão de risco por diversificação (regra dos 10):**
   - O capital destinado a operações com opções é dividido em até 10 posições por ciclo.
   - Cada posição tem tamanho aproximadamente igual (ex.: 10% do capital alocado por operação).
   - A tese é que 1 ou 2 operações vencedoras (com ganho multiplicado, típico de opções OTM que
     "explodem" de valor) cubram as perdas das demais que expirarem sem valor ou forem encerradas
     com prejuízo.
   - O agente deve **calcular e exibir** quantas posições já estão abertas, quanto capital já foi
     alocado, e alertar quando o limite de 10 for atingido.
5. O agente NUNCA deve recomendar aporte acima do tamanho de posição definido pela regra de risco,
   mesmo que o "setup" pareça muito bom.

---

## 3. Arquitetura do Sistema

```
┌─────────────────────────────────────────────────────────────┐
│                        FRONTEND (Web)                        │
│   Dashboard · Tela de Ativo · Tela de Recomendações ·        │
│   Painel de Gestão de Risco (10 posições) · Watchlist        │
└───────────────────────────┬───────────────────────────────────┘
                              │ REST/GraphQL
┌───────────────────────────▼───────────────────────────────────┐
│                      BACKEND (API)                            │
│  - Serviço de Ingestão de Dados (cotações, opções, IV)        │
│  - Serviço de Cálculo (Gregas, IV, indicadores técnicos)       │
│  - Serviço do Agente de IA (motor de recomendação)             │
│  - Serviço de Gestão de Risco (controle das 10 posições)       │
│  - Persistência (histórico de recomendações e posições)        │
└───────────────────────────┬───────────────────────────────────┘
                              │
┌───────────────────────────▼───────────────────────────────────┐
│                  FONTES DE DADOS EXTERNAS                     │
│   API de cotações/opções B3 · API de IA (LLM) para o agente    │
└─────────────────────────────────────────────────────────────┘
```

### Stack sugerida
- **Frontend:** React + TypeScript + TailwindCSS + Recharts/Chart.js (gráficos de payoff, IV smile, etc.)
- **Backend:** Python (FastAPI) — Python tem as melhores bibliotecas para cálculo de Gregas
  (ex.: `py_vollib`, `QuantLib`, `mibian`) e análise técnica (`pandas`, `ta-lib`).
- **Banco de dados:** PostgreSQL (histórico de recomendações, posições, backtests).
- **Motor de IA do agente:** chamada à API da Anthropic (Claude) para a camada de raciocínio/
  recomendação em linguagem natural, combinada com os cálculos determinísticos feitos em Python
  (as Gregas e IV **não** devem ser "estimadas" pelo LLM — devem ser calculadas por fórmulas
  matemáticas confiáveis; o LLM interpreta os números e gera a recomendação e a justificativa).
- **Deploy:** repositório no GitHub + deploy em Vercel (frontend) e Render/Railway/Fly.io (backend),
  para acesso de qualquer lugar via navegador.

---

## 4. Fonte de Dados de Mercado — Análise da Melhor Opção

**Recomendação: usar API, não scraping.** Scraping de portais financeiros tende a violar termos de
uso, quebra com frequência (mudança de layout) e não é confiável para decisões financeiras.

Opções de API a avaliar tecnicamente na fase de setup (o agente de backend deve testar
disponibilidade, custo e cobertura de cada uma antes de decidir):

| Fonte | Cobertura | Observação |
|---|---|---|
| **OpLab** | Especializada em opções B3 (Gregas, IV, liquidez já calculadas) | Melhor fit para o escopo do projeto — validar plano/preço e limites de API |
| **brapi.dev** | Cotações B3 (ações, alguns dados de opções) | Boa para cotação de ativo-objeto; validar se cobre cadeia de opções |
| **B3 (dados oficiais / Market Data)** | Dados oficiais de pregão | Mais robusto, porém contratação institucional pode ter custo mais alto |
| **Yahoo Finance / Alpha Vantage** | Cobertura ampla, mas fraca em opções BR | Usar como fallback/complemento para dados técnicos do ativo-objeto |

O time de agentes deve iniciar o projeto com uma etapa explícita de **spike técnico**: testar as
APIs candidatas, confirmar que trazem cadeia de opções (strike, vencimento, bid/ask, volume, OI,
IV) e só então integrar a definitiva.

---

## 5. Módulos Funcionais (para o backlog dos agentes)

1. **Ingestão de dados:** puxar cotação do ativo-objeto + cadeia de opções (strikes, vencimentos,
   preços, volume, open interest).
2. **Motor de cálculo:**
   - Volatilidade implícita (via Black-Scholes reverso) e volatilidade histórica.
   - Gregas: Delta, Gamma, Theta, Vega (e Rho opcional).
   - Indicadores técnicos de tendência do ativo-objeto (médias móveis, RSI, MACD — para dar
     contexto de tendência à tese de compra).
   - Métrica de liquidez da opção (volume + OI + spread bid/ask) — opções ilíquidas devem ser
     sinalizadas como risco, mesmo que "baratas".
3. **Agente de recomendação (LLM):**
   - Recebe os dados calculados (não bruscos) e gera: ativo, strike, vencimento, tipo (Call/Put),
     tese de tendência, nível de risco, e justificativa em linguagem simples.
   - Aplica sempre a regra de tamanho de posição (1/10 do capital alocado).
4. **Painel de gestão de risco:** mostra quantas das 10 posições estão abertas, capital alocado,
   e alerta quando o limite é atingido.
5. **Histórico/backtest (fase 2):** registrar recomendações passadas vs. resultado real, para medir
   a taxa de acerto da estratégia ao longo do tempo.

---

## 6. Como Estruturar os Agentes no Antigravity

Sugestão de divisão de trabalho na visão **Manager** (agentes em paralelo, workspaces separadas):

1. **Agente de Arquitetura/Backend** — modelo Claude Sonnet 4.6 ou Opus 4.6 (mais confiável para
   lógica financeira e cálculo de Gregas). Responsável por: setup do FastAPI, integração com a API
   de dados de mercado, implementação das fórmulas de Gregas/IV, motor de regras de risco.
2. **Agente de Frontend** — Gemini 3.5 Flash (rápido para boilerplate de UI). Responsável por:
   dashboard, telas de recomendação, gráficos, painel de risco.
3. **Agente de Testes** — usa o Browser Subagent do Antigravity para testar o fluxo ponta a ponta
   (abre a aplicação, valida se as recomendações aparecem corretamente, testa o limite das 10
   posições).
4. **Agente de Deploy** — configura CI/CD, repositório GitHub, deploy em Vercel/Render.

Recomenda-se pedir ao Antigravity, no primeiro prompt, para gerar o **Implementation Plan**
(artefato nativo da ferramenta) a partir deste documento, revisar esse plano antes de deixar os
agentes executarem, e só então liberar a execução em paralelo.

---

## 7. Roadmap

**Fase 1 — MVP**
- Ingestão de dados de 1 ativo por vez + cadeia de opções.
- Cálculo de Gregas e IV.
- Agente de IA gera recomendação simples (texto) para compra de Call/Put OTM.
- Painel básico de gestão de risco (contador de posições 1 a 10).

**Fase 2**
- Multiativos + watchlist.
- Gráfico de payoff da opção recomendada.
- Histórico de recomendações e acompanhamento de resultado.

**Fase 3**
- Backtest da estratégia.
- Alertas automáticos quando um setup OTM interessante surgir.

---

## 8. Avisos e Limites do Projeto

- Ferramenta de apoio à decisão — não é consultoria de investimento nem executa ordens.
- Toda recomendação deve vir com o nível de risco explícito e o disclaimer de que operações com
  opções podem levar à perda total do valor investido na opção.
- Dados de mercado devem sempre indicar a fonte e o horário da cotação (evitar decisão sobre dado
  desatualizado).
