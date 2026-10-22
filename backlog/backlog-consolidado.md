# Backlog Consolidado - TP3 - TradePilot AI


## 1. Síntese dos Aprendizados de Validação

A consolidação do backlog final reflete diretamente as evidências documentais coletadas no TP1 (`F1`–`F8`) e a validação navegacional e arquitetural conduzida no TP2:

1. **Atrito de Entrada e Evasão no Registro Manual (`F7`, Persona Rafael, TP2 Insight 1):**
   - *Evidência:* O preenchimento manual de planilhas consome mais de 25 minutos/dia e leva à evasão de mais de 80% dos traders após as duas primeiras semanas.
   - *Impacto no Backlog:* A importação automática (`RF-M1-5`) e o Onboarding Invertido ("Value First", `ADR-04`) foram elevados à prioridade máxima da jornada de ativação (`< 3 min`, `RNF-25`). Incorporou-se o suporte direto ao CSV de ordens do **Profit** (TP2, p. 28) e uma **base demonstrativa sintetizada** para usuários que ainda não possuem arquivos no primeiro acesso.

2. **Divergência entre Nota de Corretagem (PDF) e Extrato B3 (`F7`, Validação do TP2):**
   - *Evidência:* Na validação prática do TP2 com trader pessoa física, verificamos que a nota da corretora em PDF e o extrato da B3 em Excel raramente batem de primeira, consumindo horas de conciliação manual.
   - *Impacto no Backlog:* A **Lacuna 1** do TP2 formalizou o passo condicional **"Conflitos de Conciliação"** dentro de `/diario/importar` (`RF-M1-6`, `RF-M1-7`, `FN-ING-03`, `FN-ING-04`), permitindo comparar lado a lado e escolher *Ignorar duplicata*, *Sobrescrever* ou *Mesclar custos*.

3. **Desconfiança Analítica ("Número Inexplicável") (`F6`, `RNF-29`, Momento Crítico 2):**
   - *Evidência:* Métricas que divergem da memória do trader ou sem fórmula explícita destroem a credibilidade da plataforma e motivam cancelamento.
   - *Impacto no Backlog:* A **Lacuna 3** e a `ADR-02` do TP2 tornaram obrigatório o componente **"Raio-X da Fórmula"** (`FN-PLT-07`) em todo cartão de métrica numérica, além do isolamento estrito de cálculos determinísticos em código (`RNF-19`, `RNF-20`).

4. **Risco Fiscal, Silos Tributários e Prejuízos Anteriores (`F4`, Stakeholder Contador):**
   - *Evidência:* Traders misturam Day Trade (20%, sem isenção) com Swing Trade (15%, isenção até R$ 20.000) e esquecem o vencimento da DARF. Além disso, novos usuários chegam à plataforma com prejuízos acumulados de meses/anos anteriores que não constam nas notas importadas.
   - *Impacto no Backlog:* A **Lacuna 2** do TP2 adicionou a funcionalidade **"Saldo Inicial de Prejuízos Anteriores"** (`/fiscal/ajustes`, `SPEC-21`) e travas físicas/visuais de segregação de silos (`PROC-FISC`). O motor de FIIs/ETFs/Opções (`RF-M5-5`) permaneceu em **V2**, embora sua arquitetura de silo já esteja prevista no fluxo `PROC-FISC`.

5. **Ergonomia Pré-Trade Durante o Pregão (Validação do TP2):**
   - *Evidência:* Verificamos que sair da tela do Diário/gráfico para abrir a Calculadora de Risco quebrava o fluxo operacional.
   - *Impacto no Backlog:* Incorporou-se o **atalho global de teclado (`Alt+C` ou `Ctrl+K`)** invocando a Calculadora de Posição (`RF-M7-2`, `FN-RISC-02`) como gaveta lateral flutuante (*drawer*) em qualquer tela, mantendo a postura passiva exigida pela CVM (`F8`, `RNF-28`).

6. **Soberania de Dados (LGPD) e Trilha de Auditoria (`F8`, `RNF-13`, `RNF-30`):**
   - *Evidência:* Alterações manuais em operações e apurações fiscais exigem rastreabilidade imutável, e a LGPD exige exportação aberta e exclusão definitiva.
   - *Impacto no Backlog:* A **Lacuna 4** do TP2 formalizou `FN-PLT-04`, `FN-PLT-05` e `FN-PLT-06` em `/configuracoes/dados` com log *append-only*.

---

## 2. Backlog Consolidado - Escopo Inicial (MVP)

O Escopo Inicial consolidado é composto por **27 itens especificáveis** (`SPEC-01` a `SPEC-27`), que cobrem integralmente os **23 requisitos `MUST`/`SHOULD` de Escopo Inicial (EI) do TP1** acrescidos dos **4 ajustes estruturais descobertos e validados no TP2**.

| ID Spec | Título do Item | Épico / Capacidade | Origem TP1 (`RF`/`RNF`) | Função TP2 (`FN`) | Prioridade | Status de Origem |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **SPEC-01** | Cadastro, Autenticação JWT, Recuperação e 2FA Opcional | Transversal (`CAP-07`) | `RF-T-1`, `RNF-10`, `RNF-12` | `FN-PLT-01` | MUST | TP1 (EI) |
| **SPEC-02** | Planos, Cobrança e Gestão de Assinatura | Transversal (`CAP-07`) | `RF-T-2` | `FN-PLT-03` | MUST | TP1 (EI) |
| **SPEC-03** | Exportação Completa de Dados e Exclusão de Conta (LGPD) | Transversal (`CAP-07`) | `RF-T-4`, `RNF-13` | `FN-PLT-04`, `FN-PLT-05` | MUST | TP1 (EI) + TP2 Lacuna 4 |
| **SPEC-04** | Trilha de Auditoria Transacional Imutável (Append-Only) | Transversal (`CAP-07`) | `RNF-15`, `RNF-30` | `FN-PLT-06` | MUST | TP2 Lacuna 4 |
| **SPEC-05** | Onboarding Guiado Invertido ("Value First" + Base Demo) | Transversal (`CAP-07`) | `RF-T-3`, `RNF-25` | `FN-PLT-02` | MUST | TP1 (EI) + TP2 ADR-04 |
| **SPEC-06** | Ingestão Assíncrona de Extrato B3 e Ordens Profit (CSV/XLSX) | Ingestão (`CAP-01`) | `RF-M1-5`, `RNF-3`, `RNF-9` | `FN-ING-01` | MUST | TP1 (EI) + TP2 p. 28 |
| **SPEC-07** | Ingestão Assíncrona de Notas de Corretagem SINACOR (PDF) | Ingestão (`CAP-01`) | `RF-M1-5`, `RNF-3`, `RNF-9` | `FN-ING-02` | MUST | TP1 (EI) |
| **SPEC-08** | Motor de Deduplicação Idempotente por Hash | Ingestão (`CAP-01`) | `RF-M1-7`, `RNF-9` | `FN-ING-03` | SHOULD | TP1 (EI) |
| **SPEC-09** | Wizard de Conciliação Assistida e Resolução de Conflitos | Ingestão (`CAP-01`) | `RF-M1-6`, `RF-M1-7`, `RNF-30` | `FN-ING-04` | MUST | TP1 (EI) + TP2 Lacuna 1 |
| **SPEC-10** | Cadastro Manual Completo de Trade | Diário (`CAP-02`) | `RF-M1-1`, `RNF-1` | `FN-DIA-01` | MUST | TP1 (EI) |
| **SPEC-11** | Anexo Seguro de Capturas de Tela de Setup | Diário (`CAP-02`) | `RF-M1-2`, `RNF-10` | `FN-DIA-02` | MUST | TP1 (EI) |
| **SPEC-12** | Contextualizador Operacional (Setup, Plano e Motivos) | Diário (`CAP-02`) | `RF-M1-3` | `FN-DIA-03` | MUST | TP1 (EI) |
| **SPEC-13** | Registro Psico-Emocional Pré e Pós-Operação | Diário (`CAP-02`) | `RF-M1-4` | `FN-DIA-04` | MUST | TP1 (EI) |
| **SPEC-14** | Motor Determinístico de Métricas Core (Acerto, Payoff, PF, Expectativa, Streaks, Drawdown) | Analytics (`CAP-03`) | `RF-M2-1`, `RNF-19`, `RNF-20`, `RNF-29` | `FN-ANA-01`, `FN-ANA-02`, `FN-ANA-03` | MUST | TP1 (EI) |
| **SPEC-15** | Gerador de Curva de Capital (Equity Curve) | Analytics (`CAP-03`) | `RF-M2-2`, `RNF-2` | `FN-ANA-04` | MUST | TP1 (EI) |
| **SPEC-16** | Segmentador Multidimensional (Horário, Dia, Ativo, Setup) | Analytics (`CAP-03`) | `RF-M2-3`, `RNF-1` | `FN-ANA-05` | MUST | TP1 (EI) |
| **SPEC-17** | Sintetizador de Insights Automáticos Determinísticos | Analytics (`CAP-03`) | `RF-M2-4`, `RNF-28`, `RNF-29` | `FN-ANA-06` | SHOULD | TP1 (EI) |
| **SPEC-18** | Dashboard Executivo e Calendário Operacional Dinâmico | Síntese (`CAP-06`) | `RF-M8-1`, `RNF-2`, `RNF-23`, `RNF-24` | `FN-DASH-01`, `FN-DASH-02` | MUST | TP1 (EI) |
| **SPEC-19** | Gerenciador de Filtro Temporal Global | Síntese (`CAP-06`) | `RF-M8-2` | `FN-DASH-03` | MUST | TP1 (EI) |
| **SPEC-20** | Componente Universal "Raio-X da Fórmula" | Transversal (`CAP-03`/`CAP-06`) | `RNF-29` | `FN-PLT-07` | MUST | TP2 Lacuna 3 / ADR-02 |
| **SPEC-21** | Gestão de Parâmetros e Tetos de Risco (Operação, Dia, Semana, Mês) | Risco (`CAP-05`) | `RF-M7-1` | `FN-RISC-01` | MUST | TP1 (EI) |
| **SPEC-22** | Calculadora Passiva de Tamanho de Posição com Atalho Global (`Alt+C`/`Ctrl+K`) | Risco (`CAP-05`) | `RF-M7-2`, `RNF-28` | `FN-RISC-02` | MUST | TP1 (EI) + Retorno TP2 |
| **SPEC-23** | Segregador Tributário (Day Trade vs Swing Trade) e Isenção R$ 20k | Fiscal (`CAP-04`) | `RF-M5-1`, `RNF-20` | `FN-FISC-01`, `FN-FISC-02` | MUST | TP1 (EI) |
| **SPEC-24** | Saldo Inicial de Prejuízos Anteriores (`/fiscal/ajustes`) | Fiscal (`CAP-04`) | `RF-M5-2`, `RNF-30` | `FN-FISC-03` | MUST | TP2 Lacuna 2 |
| **SPEC-25** | Compensador de Prejuízos por Silo e Apurador de Imposto / IRRF | Fiscal (`CAP-04`) | `RF-M5-1`, `RF-M5-2`, `RNF-19`, `RNF-20` | `FN-FISC-03`, `FN-FISC-04` | MUST | TP1 (EI) |
| **SPEC-26** | Gerador de DARF Código 6015 (Valor ≥ R$ 10,00, Vencimento e PIX) | Fiscal (`CAP-04`) | `RF-M5-3`, `RNF-7` | `FN-FISC-05` | MUST | TP1 (EI) |
| **SPEC-27** | Emissor de Relatório Mensal e Memória de Cálculo Auditável em PDF | Fiscal (`CAP-04`) | `RF-M5-4`, `RNF-29` | `FN-FISC-06` | MUST | TP1 (EI) |

---

## 3. Backlog Futuro

Os itens abaixo permanecem fora do ciclo inicial de construção, conforme critérios de viabilidade, dependência de dados externos (`F1`, licença B3) e necessidade de massa crítica de histórico do usuário:

- **V2 — Expansão Analítica e Fiscal:**
  - `RF-M1-8` (Consolidar execuções parciais), `RF-M1-9` (Edição em massa com auditoria).
  - `RF-M2-5`, `RF-M2-6`, `RF-M2-7` (Sharpe, Sortino, MAE/MFE, R-multiple, filtros combináveis, comparação de períodos).
  - `RF-M5-5`, `RF-M5-6`, `RF-M5-7` (Motor fiscal de FIIs 20%, ETFs 15% e Opções com silo próprio; Relatório Anual IRPF; Exportação GCAP).
  - `RF-M6-1`, `RF-M6-2`, `RF-M6-3` (Psicologia avançada e correlação emocional profunda).
  - `RF-M7-3` (Alertas comportamentais de overtrading e revenge trading).
  - `RF-M8-3` (Mapas de calor), `RF-M9-1`, `RF-M9-2` (Relatórios com narrativa IA e PDF), `RF-M10-1`, `RF-M10-2` (Playbooks).
  - `RF-M3-1`, `RF-M3-2`, `RF-M3-3` (Scanner de mercado — condicionado à contratação de licença de dados B3).
  - `RF-M4-1` a `RF-M4-4` (Copiloto IA pós-trade com histórico real e restrição CVM por código).
  - `RF-T-5` (Notificações push/e-mail).

- **V3 — Escala e Ecossistema (COULD):**
  - `RF-M1-10` (Integração direta com corretoras parceiras), `RF-M3-4`, `RF-M3-5`, `RF-M4-5`, `RF-M4-6`, `RF-M5-8`, `RF-M6-4`, `RF-M7-4`, `RF-M9-3` (Relatório para mentor/mesa B2B), `RF-M10-3`, `RF-T-6` (API pública e webhooks).

---

## 4. Escopo Negativo (Restrições Permanentes)

Por decisão regulatória (`F8` — CVM / Receita Federal) e posicionamento de produto, o TradePilot AI **nunca** implementará:
1. Execução ou roteamento de ordens de compra e venda (não é corretora).
2. Recomendação de ativos, preços-alvo ou sinais de entrada/saída por IA (vedação CVM).
3. Transmissão automática da declaração de IRPF ou pagamento bancário direto da DARF (calcula, orienta e emite memória auditável).
4. Substituição do contador ou consultor tributário profissional.
5. Plataforma de gráficos em tempo real concorrente do Profit/TradingView.
