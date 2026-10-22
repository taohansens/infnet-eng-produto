# Cronograma Inicial de Sprints — TradePilot AI

## 1. Critérios Explícitos de Priorização

A ordenação dos itens nas Sprints não é arbitrária; ela aplica uma matriz ponderada de **4 critérios objetivos** derivados das evidências do TP1 e da arquitetura do TP2:

| Critério | Peso | Regra de Decisão | Evidência de Base |
| :--- | :---: | :--- | :--- |
| **1. Dependência Estrutural e Segurança (Fundação)** | 30% | Autenticação, isolamento multi-tenant no banco (RLS), criptografia e trilha de auditoria devem existir antes de qualquer dado financeiro ou fiscal ser persistido. | `F8` (LGPD), `RNF-11`, `RNF-30`, `CAP-07` |
| **2. Redução do Atrito de Entrada (Jornada de Adoção)** | 25% | O fluxo `Importar → Visualizar → Confiar → Cumprir obrigação` exige que a ingestão de dados reais funcione antes da camada analítica e fiscal. | `F7`, Persona Rafael, `ADR-04` |
| **3. Mitigação de Risco Crítico (Confiança & Fiscal)** | 25% | Módulos com risco financeiro/jurídico (cálculo fiscal, deduplicação, Raio-X da Fórmula) exigem testes determinísticos (*golden tests*) antes da liberação. | `F4`, `RNF-19`, `RNF-20`, `RNF-29` |
| **4. Valor Percebido Imediato vs. Esforço** | 20% | Entregas que geram o "Momento Wow" em `< 3 min` (Dashboard + Curva de Capital + Calculadora com atalho) são priorizadas sobre recursos secundários de V2. | `RNF-25`, Retorno Avaliador TP2 |

---

## 2. Visão Geral do Cronograma (5 Sprints — 10 Semanas)

| Sprint | Duração | Tema / Objetivo da Sprint | Itens Alocados (`SPEC-xx`) | Critério de Saída (Milestone) |
| :--- | :---: | :--- | :--- | :--- |
| **Sprint 1** | Sem 1–2 | **Fundação, Segurança, LGPD e Assinaturas** | `SPEC-01`, `SPEC-02`, `SPEC-03`, `SPEC-04` | Usuário autentica com JWT/2FA, conta isolada por RLS no PostgreSQL, log *append-only* ativo e exportação/exclusão LGPD testadas. |
| **Sprint 2** | Sem 3–4 | **Ingestão Assíncrona, Deduplicação e Onboarding** | `SPEC-05`, `SPEC-06`, `SPEC-07`, `SPEC-08`, `SPEC-09` | Fluxo completo de upload (Extrato B3, CSV Profit, Nota PDF SINACOR), detecção de duplicatas por hash e Wizard de Conciliação sem travar UI (`≤ 30s` para 300 linhas). |
| **Sprint 3** | Sem 5–6 | **Diário Operacional, Analytics e Dashboard Executivo** | `SPEC-10`, `SPEC-11`, `SPEC-12`, `SPEC-13`, `SPEC-14`, `SPEC-15`, `SPEC-16`, `SPEC-17`, `SPEC-18`, `SPEC-19` | Diário completo com anexos/emoções, motor de métricas determinístico (`≥ 90%` cobertura), Dashboard `< 2s` p95 e Filtro Temporal Global. |
| **Sprint 4** | Sem 7–8 | **Confiança Numérica (Raio-X) e Governança de Risco** | `SPEC-20`, `SPEC-21`, `SPEC-22` | Raio-X da Fórmula ativo em todas as métricas (`RNF-29`), limites de risco configuráveis e Calculadora de Posição passiva via atalho global (`Alt+C`/`Ctrl+K`). |
| **Sprint 5** | Sem 9–10 | **Motor Fiscal B3, Compensação e DARF 6015** | `SPEC-23`, `SPEC-24`, `SPEC-25`, `SPEC-26`, `SPEC-27` | Silos Day Trade (20%) e Swing Trade (15%, isenção R$ 20k), Saldo Inicial de Prejuízos, DARF 6015 e Memória PDF auditável validados por *golden tests*. |

---

## 3. Detalhamento por Sprint

### Sprint 1 — Fundação, Segurança, LGPD e Assinaturas (Semanas 1–2)
- **Meta:** Estabelecer a espinha dorsal multi-tenant segura e aderente à LGPD antes de ingerir dados sensíveis.
- **Specs Alocadas:**
  - `SPEC-01`: Autenticação, JWT, Recuperação de Senha e 2FA Opcional (`RF-T-1`)
  - `SPEC-02`: Planos, Cobrança e Gestão de Assinatura (`RF-T-2`)
  - `SPEC-03`: Exportação Completa ZIP (JSON/CSV) e Exclusão Definitiva LGPD (`RF-T-4`)
  - `SPEC-04`: Trilha de Auditoria Transacional Imutável (`RNF-30`)
- **Dependências:** Nenhuma (Sprint inicial; provisiona schema PostgreSQL com Row-Level Security e Object Storage).
- **Portão de Qualidade:** Teste automatizado de vazamento entre contas (`RNF-11`) e pipeline CI obrigatório (`RNF-21`).

### Sprint 2 — Ingestão Assíncrona, Deduplicação e Onboarding (Semanas 3–4)
- **Meta:** Eliminar o maior ponto de evasão da jornada (registro manual) entregando o "Momento Wow" em `< 3 min`.
- **Specs Alocadas:**
  - `SPEC-06`: Parser Assíncrono de Extrato B3 e Ordens Profit CSV/XLSX (`RF-M1-5`)
  - `SPEC-07`: Parser Assíncrono de Notas de Corretagem PDF SINACOR (`RF-M1-5`)
  - `SPEC-08`: Motor de Deduplicação Idempotente por Hash (`RF-M1-7`)
  - `SPEC-09`: Wizard de Conciliação Assistida e Resolução de Conflitos (`RF-M1-6`, TP2 Lacuna 1)
  - `SPEC-05`: Onboarding Guiado Invertido + Base Demonstrativa Sintetizada (`RF-T-3`, `ADR-04`)
- **Dependências:** `SPEC-01` (usuário autenticado) e `SPEC-04` (auditoria de lotes).
- **Portão de Qualidade:** Benchmark de importação (`300 linhas ≤ 30s`, `RNF-3`) e testes de idempotência (`RNF-9`).

### Sprint 3 — Diário Operacional, Analytics e Dashboard Executivo (Semanas 5–6)
- **Meta:** Transformar operações importadas ou manuais em inteligência visual e estatística imediata.
- **Specs Alocadas:**
  - `SPEC-10`, `SPEC-11`, `SPEC-12`, `SPEC-13`: Diário Manual, Anexos de Setup, Contextualizador e Monitor Psico-Emocional (`RF-M1-1` a `RF-M1-4`)
  - `SPEC-14`, `SPEC-15`, `SPEC-16`, `SPEC-17`: Motor Determinístico de Métricas, Curva de Capital, Segmentador Multidimensional e Insights Determinísticos (`RF-M2-1` a `RF-M2-4`)
  - `SPEC-18`, `SPEC-19`: Dashboard Executivo, Calendário Operacional e Filtro Temporal Global (`RF-M8-1`, `RF-M8-2`)
- **Dependências:** `SPEC-06`–`SPEC-09` (base de trades persistida e saneada).
- **Portão de Qualidade:** Cobertura de testes `≥ 90%` no módulo Analytics (`RNF-19`), latência de leitura `≤ 400 ms` p95 (`RNF-1`) e carga do Dashboard `≤ 2 s` com 10.000 trades (`RNF-2`).

### Sprint 4 — Confiança Numérica (Raio-X) e Governança de Risco (Semanas 7–8)
- **Meta:** Blindar a confiança nos números exibidos e entregar governança pré-trade sem violar regras da CVM.
- **Specs Alocadas:**
  - `SPEC-20`: Componente Universal "Raio-X da Fórmula" (`FN-PLT-07`, `RNF-29`)
  - `SPEC-21`: Gestão de Parâmetros e Tetos de Risco (`RF-M7-1`)
  - `SPEC-22`: Calculadora Passiva de Tamanho de Posição com Atalho Global `Alt+C`/`Ctrl+K` (`RF-M7-2`)
- **Dependências:** `SPEC-14` e `SPEC-18` (métricas e shell global com atalhos).
- **Portão de Qualidade:** Verificação de acessibilidade WCAG 2.1 AA nos modais/drawers (`RNF-24`) e auditoria de conformidade CVM (zero linguagem de recomendação, `RNF-28`).

### Sprint 5 — Motor Fiscal B3, Compensação e DARF 6015 (Semanas 9–10)
- **Meta:** Entregar o principal diferencial competitivo local e alavanca de retenção mensal.
- **Specs Alocadas:**
  - `SPEC-23`: Segregador Tributário (Day Trade vs Swing Trade) e Verificador de Isenção R$ 20k (`RF-M5-1`)
  - `SPEC-24`: Cadastro Auditável de Saldo Inicial de Prejuízos Anteriores (`RF-M5-2`, TP2 Lacuna 2)
  - `SPEC-25`: Compensador de Prejuízos por Silo e Apurador de Imposto / IRRF (`RF-M5-1`, `RF-M5-2`)
  - `SPEC-26`: Gerador de DARF Código 6015 com Linha Digitável / PIX (`RF-M5-3`)
  - `SPEC-27`: Emissor de Relatório Mensal e Memória de Cálculo Auditável em PDF (`RF-M5-4`)
- **Dependências:** `SPEC-06`–`SPEC-09` (operações + taxas/IRRF das notas) e `SPEC-04` (auditoria de ajustes fiscais).
- **Portão de Qualidade:** Suíte completa de *Golden Tests* (`RNF-20`) cobrindo casos conhecidos da Receita Federal (isenção de R$ 20k, limite mínimo de R$ 10,00 para DARF, compensação estrita por silo) e revisão por contador antes do lançamento.

---

## 4. Definition of Done (DoD) Global das Sprints

Um item `SPEC-xx` só é considerado **Concluído (Done)** quando:
1. Todos os cenários Gherkin (`Dado / Quando / Então`) da especificação passam em testes automatizados.
2. O contrato de dados (JSON Schema / DTO) e a documentação de API estão atualizados (`RNF-22`).
3. A cobertura de testes atinge `≥ 90%` em módulos de cálculo (Analytics/Fiscal) e `≥ 75%` no geral (`RNF-19`).
4. O pipeline de CI (lint, tipagem, testes e build) passa sem erros (`RNF-21`).
5. Para itens com interface: responsividade, idioma PT-BR, modo escuro (`RNF-23`) e acessibilidade AA (`RNF-24`) verificados.
