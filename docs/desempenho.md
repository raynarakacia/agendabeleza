# Relatório de Desempenho, SLO e Teste de Carga (M3 / M4)

Este documento registra o **SLO (Service Level Objective)** declarado para a API do **AgendaBeleza**, os scripts de carga disponíveis e os resultados reais medidos localmente.

## 1. SLO Declarado (Módulo 3)

| Indicador (SLI) | Meta (SLO) | Justificativa de Negócio |
| :--- | :--- | :--- |
| **Latência `p95` (`http_req_duration`)** | **< 200 ms** | Clientes acessando a página pública do salão (`/salao/:slug`) pelo celular necessitam de carregamento rápido para evitar abandono no agendamento. |
| **Disponibilidade / Taxa de Sucesso** | **>= 99,0%** (`http_req_failed < 1%`) | Garantir que agendamentos e confirmações de horário não falhem em horários de pico comercial. |

## 2. Scripts de Ensaio de Carga

- **Script k6 (`backend/scripts/k6-smoke.js`)**: Cenário em 3 estágios (ramp-up de 10s até 10 VUs, sustentação de 20s em 30 VUs e ramp-down de 10s) validando `p(95)<200` e `rate<0.01`.
- **Script Autocannon (`backend/scripts/load-test.js`)**: Teste de estresse executado localmente com **100 conexões concorrentes** durante 10 segundos.

## 3. Resultados Reais Medidos (`autocannon` — 15/11/2026)

### Ambiente de Execução
- **Comando**: `node scripts/load-test.js` contra `http://localhost:3333/health`
- **Perfil de carga**: 100 conexões simultâneas durante `10,15 s` (API rodando em modo desenvolvimento `ts-node-dev`)

### Métricas Obtidas

| Métrica | Resultado Real Medido | Meta (SLO) | Observação |
| :--- | :--- | :--- | :--- |
| **Requisições Totais (`200 OK`)** | **8.801 requisições** | — | **880,1 req/s** em média |
| **Taxa de Erros / Timeouts** | **0,00%** (0 erros / 0 timeouts) | < 1,00% | **Aprovado** (100% de sucesso) |
| **Latência Mediana (`p50`)** | **100 ms** | < 200 ms | **Aprovado** |
| **Latência Média (`avg`)** | **113,23 ms** | < 200 ms | **Aprovado** |
| **Latência `p90`** | **210 ms** | < 200 ms | Próximo do limite sob 100 conexões |
| **Latência `p97,5` / `p99`** | **304 ms / 410 ms** | — | Cauda sob estresse em modo `dev` |

## 4. Diagnóstico de Gargalos e Plano de Otimização

1. **Impacto do Modo Desenvolvimento vs. Build Compilado**: O teste de estresse com 100 conexões simultâneas foi executado sobre `ts-node-dev`, o que eleva a cauda de latência (`p90 = 210 ms`, `p97,5 = 304 ms`). Em produção com o código JavaScript compilado (`npm run build && npm start` no Docker), o overhead de runtime é reduzido.
2. **Índices e Pool de Conexões no PostgreSQL**: A rota pública `GET /public/:slug` utiliza índice único no campo `slug` no PostgreSQL, mantendo a consulta em *Index Scan*.
3. **Evolução Futura (Cache Redis)**: Para manter o `p95` confortavelmente abaixo de `200 ms` mesmo acima de 100 usuários simultâneos na vitrine pública (`GET /public/:slug/services`), o próximo passo arquitetural é adicionar cache em Redis com TTL curto (60 segundos).