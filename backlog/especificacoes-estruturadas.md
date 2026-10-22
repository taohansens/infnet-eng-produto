# Especificações Estruturadas do Backlog (Spec-Driven) - TradePilot AI

## Sumário de Rastreabilidade por Módulo

- **Módulo Transversal & LGPD (`CAP-07`):** `SPEC-01` a `SPEC-05`
- **Módulo de Ingestão & Conciliação (`CAP-01`):** `SPEC-06` a `SPEC-09`
- **Módulo Diário de Trade (`CAP-02`):** `SPEC-10` a `SPEC-13`
- **Módulo Trading Analytics (`CAP-03`):** `SPEC-14` a `SPEC-17`
- **Módulo Dashboard & Transparência (`CAP-06` / `CAP-03`):** `SPEC-18` a `SPEC-20`
- **Módulo Gestão de Risco (`CAP-05`):** `SPEC-21` a `SPEC-22`
- **Módulo Gestão Fiscal B3 (`CAP-04`):** `SPEC-23` a `SPEC-27`

---

## 1. Módulo Transversal & LGPD (`CAP-07`)

### `SPEC-01` — Cadastro, Autenticação JWT, Recuperação de Senha e 2FA Opcional
- **Rastreabilidade:** `RF-T-1` | `FN-PLT-01` | `RNF-10`, `RNF-11`, `RNF-12` | Fonte: `F8` | Rota: `/login`, `/cadastro`, `/recuperar-senha`
- **História de Usuário:** Como trader pessoa física, quero criar uma conta segura com opção de 2FA para proteger meu histórico operacional e fiscal contra acessos não autorizados.
- **Pré-condições:** Comunicação via TLS 1.2+ (`RNF-10`).
- **Regras de Negócio:**
  1. Senhas devem ter no mínimo 10 caracteres e serem armazenadas com hash forte (ex.: Argon2id/bcrypt) (`RNF-12`).
  2. Sessão autenticada emite token JWT de curta duração + refresh token rotacionável.
  3. Toda requisição autenticada deve propagar `user_id` para a política de isolamento por linha (RLS) no banco (`RNF-11`).
  4. 2FA via TOTP (aplicativo autenticador) é opcional no cadastro e ativável em `/configuracoes`.
- **Contrato de Dados (JSON):**
  ```json
  // POST /api/v1/auth/register -> Request
  { "name": "string", "email": "string (RFC 5322)", "password": "string (min 10)", "lgpd_consent": true }
  // Response 201
  { "user_id": "uuid", "access_token": "jwt", "expires_in": 900, "requires_onboarding": true }
  ```
- **Critérios de Aceitação:**
  - **Dado** um visitante na tela `/cadastro` com e-mail válido e aceite de termo LGPD, **Quando** enviar o formulário, **Então** a conta é criada com hash seguro, `requires_onboarding = true` e o usuário é redirecionado para `/onboarding`.
  - **Dado** um usuário com 2FA ativo, **Quando** informar senha válida no `/login`, **Então** o sistema solicita o código TOTP de 6 dígitos antes de emitir o JWT final.

---

### `SPEC-02` — Planos, Cobrança e Gestão de Assinatura
- **Rastreabilidade:** `RF-T-2` | `FN-PLT-03` | `RNF-26` | Fonte: `F5` | Rota: `/configuracoes`
- **História de Usuário:** Como assinante, quero visualizar meu plano, fazer upgrade/downgrade e gerenciar minha cobrança para controlar meu acesso aos recursos.
- **Regras de Negócio:**
  1. Permissões de funcionalidades são derivadas do `plan_status` (`trial`, `active`, `past_due`, `canceled`).
  2. Alterações de status ocorrem de forma idempotente via webhook assinado do gateway de pagamento.
  3. Em caso de `canceled` ou inadimplência após período de carência, o acesso de escrita é bloqueado, mas a exportação de dados LGPD (`SPEC-03`) permanece liberada.
- **Contrato de Dados (JSON):**
  ```json
  // GET /api/v1/billing/subscription -> Response 200
  { "plan_code": "pro_monthly", "status": "active", "current_period_end": "2026-12-22T00:00:00Z", "features": ["import_pdf", "fiscal_darf"] }
  ```
- **Critérios de Aceitação:**
  - **Dado** um webhook de pagamento confirmado com assinatura válida, **Quando** recebido pelo endpoint de billing, **Então** o plano do usuário é atualizado para `active` de forma idempotente e registrado na trilha de auditoria.

---

### `SPEC-03` — Exportação Completa de Dados e Exclusão Definitiva (LGPD)
- **Rastreabilidade:** `RF-T-4` | `FN-PLT-04`, `FN-PLT-05` | `RNF-13`, `RNF-30` | Fonte: `F8` | Lacuna 4 (TP2) | Rota: `/configuracoes/dados`
- **História de Usuário:** Como titular dos dados, quero exportar todo o meu histórico em formato aberto e poder excluir definitivamente minha conta para exercer meus direitos da LGPD.
- **Regras de Negócio:**
  1. A exportação gera assincronamente um pacote `.zip` contendo arquivos `JSON` e `CSV` de trades, notas, anexos, parâmetros de risco e apurações fiscais.
  2. A exclusão exige confirmação explícita dupla (modal de confirmação + digitação de frase ou senha).
  3. Ao confirmar exclusão, dados pessoais e financeiros são removidos/anonimizados irreversivelmente, preservando apenas registro mínimo anonimizado na trilha de auditoria quando exigido por obrigação legal (`RNF-30`).
- **Contrato de Dados (JSON):**
  ```json
  // POST /api/v1/privacy/export -> Response 202
  { "job_id": "uuid", "status": "processing", "estimated_seconds": 30 }
  // DELETE /api/v1/privacy/account -> Request
  { "confirmation_phrase": "EXCLUIR MINHA CONTA", "password": "string" }
  ```
- **Critérios de Aceitação:**
  - **Dado** um usuário autenticado em `/configuracoes/dados`, **Quando** clicar em "Exportar Meus Dados", **Então** um job assíncrono gera um `.zip` com JSON/CSV e disponibiliza o link temporário de download.
  - **Dado** um usuário confirmando a exclusão no modal duplo, **Quando** submeter a frase e senha corretas, **Então** a sessão é encerrada, os dados do titular são eliminados e o evento é auditado.

---

### `SPEC-04` — Trilha de Auditoria Transacional Imutável (Append-Only)
- **Rastreabilidade:** `RNF-15`, `RNF-30` | `FN-PLT-06` | Fonte: `F8`, `F4` | Lacuna 4 (TP2) | Rota: `/configuracoes/dados`
- **História de Usuário:** Como usuário e auditor, quero que toda criação, edição manual ou exclusão de operação e ajuste fiscal fique registrada de forma imutável para garantir integridade perante a Receita Federal.
- **Regras de Negócio:**
  1. Tabela `audit_logs` é estritamente *append-only* (sem `UPDATE` ou `DELETE` na camada de aplicação).
  2. Eventos auditados obrigatoriamente: `TRADE_MANUAL_CREATE`, `TRADE_EDIT`, `TRADE_DELETE`, `CONCILIATION_MERGE`, `FISCAL_PRIOR_LOSS_SET`, `DATA_EXPORT`, `ACCOUNT_DELETE`.
  3. Cada registro armazena `actor_id`, `action`, `entity_type`, `entity_id`, `before_snapshot`, `after_snapshot`, `created_at_utc`.
- **Contrato de Dados (JSON):**
  ```json
  { "audit_id": "uuid", "action": "FISCAL_PRIOR_LOSS_SET", "entity_id": "uuid", "before": { "day_trade": 0 }, "after": { "day_trade": 1500.00 }, "created_at": "2026-11-22T19:00:00Z" }
  ```
- **Critérios de Aceitação:**
  - **Dado** uma edição manual no preço de saída de um trade, **Quando** a transação for persistida, **Então** um registro imutável com estado anterior e posterior é gravado atomicamente na `audit_logs`.

---

### `SPEC-05` — Onboarding Guiado Invertido ("Value First" + Base Demonstrativa)
- **Rastreabilidade:** `RF-T-3` | `FN-PLT-02` | `RNF-25` | `ADR-04` | Fonte: `F7` | Estágio 5 | Rota: `/onboarding`
- **História de Usuário:** Como novo usuário, quero subir meu primeiro extrato logo após o cadastro (ou explorar uma base demonstrativa) para ver meu diagnóstico real em menos de 3 minutos sem formulários longos.
- **Regras de Negócio:**
  1. Fluxo obrigatório em 3 passos: (1) Upload de Extrato B3/Profit/PDF ou escolha de "Explorar Base Demonstrativa"; (2) Prévia e Conciliação Rápida; (3) Primeiro Diagnóstico Revelado (`/dashboard`).
  2. Tempo total alvo do fluxo `≤ 5 min` (`RNF-25`), com meta de `< 3 min` (`ADR-04`).
  3. A base demonstrativa é marcada com `is_demo = true` e pode ser limpa em 1 clique quando o usuário importar seus dados reais.
- **Contrato de Dados (JSON):**
  ```json
  // POST /api/v1/onboarding/demo-seed -> Response 201
  { "seeded_trades": 45, "is_demo": true, "redirect_to": "/dashboard" }
  ```
- **Critérios de Aceitação:**
  - **Dado** um usuário recém-cadastrado sem histórico em `/onboarding`, **Quando** selecionar "Usar Base Demonstrativa", **Então** o sistema popula uma amostra sintética marcada como demo e exibe o Dashboard em menos de 5 segundos.
  - **Dado** um usuário que faz upload de arquivo B3 no Passo 1, **Quando** concluir o Passo 2, **Então** o sistema marca `onboarding_completed = true` e apresenta o primeiro diagnóstico no Passo 3.

---

## 2. Módulo de Ingestão & Conciliação (`CAP-01`)

### `SPEC-06` — Ingestão Assíncrona de Extrato B3 e Ordens Profit (CSV/XLSX)
- **Rastreabilidade:** `RF-M1-5` | `FN-ING-01` | `RNF-3`, `RNF-9`, `ADR-01` | Fonte: `F7`, `F5` | TP2 p. 28 | Estágio 5 | Rota: `/diario/importar`
- **História de Usuário:** Como trader, quero importar arquivos CSV/XLSX do extrato da B3 ("Negociação de Ativos") ou exportação de ordens do Profit para evitar digitação manual.
- **Regras de Negócio:**
  1. Upload processado assincronamente via fila/worker (`ADR-01`), retornando `job_id` imediato e atualizando progresso via WebSocket ou polling.
  2. Desempenho: `≤ 30 s` para até 300 linhas e `≤ 5 min` para até 20.000 linhas (`RNF-3`).
  3. Normaliza ticker (ex.: `WINZ26`, `PETR4`), data/hora em UTC/BRT, lado (`BUY`/`SELL`), quantidade, preço unitário e origem (`B3_CSV`, `PROFIT_CSV`).
  4. Erros de layout exibem mensagem clara em PT-BR sem jargão técnico (ex.: *"Inconsistência no Extrato da Corretora na linha 14"*).
- **Contrato de Dados (JSON):**
  ```json
  // POST /api/v1/ingestion/jobs (multipart/form-data: file, source_hint) -> Response 202
  { "job_id": "uuid", "status": "queued", "file_name": "negociacao_b3.xlsx" }
  ```
- **Critérios de Aceitação:**
  - **Dado** um arquivo CSV válido do Profit com 300 linhas, **Quando** enviado em `/diario/importar`, **Então** o sistema retorna `202 Accepted` com `job_id` e conclui o parsing em staging em `≤ 30 s`.
  - **Dado** um arquivo corrompido ou com colunas obrigatórias ausentes, **Quando** processado pelo worker, **Então** o job falha com mensagem legível indicando a linha e coluna divergentes.

---

### `SPEC-07` — Ingestão Assíncrona de Notas de Corretagem SINACOR (PDF)
- **Rastreabilidade:** `RF-M1-5` | `FN-ING-02` | `RNF-3`, `RNF-9`, `ADR-01` | Fonte: `F7` | Estágio 5 | Rota: `/diario/importar`
- **História de Usuário:** Como trader, quero importar notas de corretagem em PDF (padrão SINACOR) para capturar automaticamente operações, taxas da B3, corretagem e IRRF retido na fonte.
- **Regras de Negócio:**
  1. Extrai cabeçalho (corretora, data do pregão, número da nota), negócios executados e rodapé financeiro (taxa de liquidação, emolumentos, corretagem, ISS, IRRF Day Trade 1% / Swing 0,005%).
  2. Rateia custos operacionais proporcionalmente ao volume financeiro de cada negócio da nota, preservando a soma exata dos centavos.
  3. Executa em worker assíncrono em staging antes da confirmação pelo usuário (`ADR-01`).
- **Contrato de Dados (JSON):**
  ```json
  // Staging item extraído de PDF
  { "broker": "XP", "trade_date": "2026-10-09", "ticker": "WDOX26", "side": "BUY", "qty": 2, "price": 5450.50, "fees": 2.40, "irrf": 0.00 }
  ```
- **Critérios de Aceitação:**
  - **Dado** um PDF de nota padrão SINACOR com operações e IRRF retido, **Quando** processado pelo worker, **Então** o sistema extrai todas as linhas, soma de taxas e valor exato de IRRF para a etapa de prévia/conciliação.

---

### `SPEC-08` — Motor de Deduplicação Idempotente por Hash
- **Rastreabilidade:** `RF-M1-7` | `FN-ING-03` | `RNF-9` | Fonte: `F7` | Estágio 5 | Rota: `/diario/importar`
- **História de Usuário:** Como trader que importa tanto o extrato da B3 quanto a nota da corretora, quero que o sistema detecte operações duplicadas para não distorcer meu resultado e meu imposto.
- **Regras de Negócio:**
  1. Calcula hash SHA-256 determinístico por negócio normalizado: `hash(user_id, ticker, executed_at_minute, side, qty, price_cents)`.
  2. Compara o hash dos itens em staging contra operações já persistidas do mesmo `user_id`.
  3. Classifica cada item do lote em: `NEW` (sem conflito), `EXACT_DUPLICATE` (mesmo hash e mesmos custos) ou `CONFLICT` (mesmo negócio base com divergência em taxas/corretagem/IRRF).
- **Contrato de Dados (JSON):**
  ```json
  { "staging_item_id": "uuid", "dedup_status": "CONFLICT", "matched_trade_id": "uuid", "diff_fields": ["fees", "irrf"] }
  ```
- **Critérios de Aceitação:**
  - **Dado** uma operação já importada via extrato B3 e posteriormente reimportada via nota PDF da corretora com taxas adicionais, **Quando** o motor de deduplicação rodar, **Então** o item é marcado como `CONFLICT` apontando os campos divergentes (`fees`, `irrf`).

---

### `SPEC-09` — Wizard de Conciliação Assistida e Resolução de Conflitos
- **Rastreabilidade:** `RF-M1-6`, `RF-M1-7` | `FN-ING-04` | `RNF-30`, `ADR-01`, `ADR-03` | Fonte: `F7` | Lacuna 1 (TP2) | Rota: `/diario/importar`
- **História de Usuário:** Como trader, quero revisar conflitos entre extrato B3 e nota de corretagem lado a lado antes de salvar no diário para garantir precisão nos meus números.
- **Regras de Negócio:**
  1. Se o lote tiver `0` itens `CONFLICT`/`EXACT_DUPLICATE`, exibe prévia resumida para confirmação em 1 clique.
  2. Se houver conflitos, exibe o passo condicional obrigatório **"Conflitos de Conciliação"** com comparação lado a lado (`Existente` vs `Importado`) e 3 ações explícitas: `IGNORE` (ignorar duplicata), `OVERWRITE` (sobrescrever existente) ou `MERGE_COSTS` (manter trade e incorporar taxas/IRRF da nota).
  3. A persistência final do lote ocorre em transação atômica única (`RNF-9`) com registro em `audit_logs` (`SPEC-04`).
- **Contrato de Dados (JSON):**
  ```json
  // POST /api/v1/ingestion/jobs/{job_id}/commit -> Request
  { "resolutions": [{ "staging_item_id": "uuid", "action": "MERGE_COSTS" }] }
  // Response 200
  { "inserted": 12, "merged": 3, "ignored": 1 }
  ```
- **Critérios de Aceitação:**
  - **Dado** um lote em staging com 2 conflitos detectados, **Quando** o usuário escolher `MERGE_COSTS` para ambos e confirmar, **Então** as taxas e o IRRF são incorporados às operações existentes em transação atômica sem duplicar o resultado bruto.

---

## 3. Módulo Diário de Trade (`CAP-02`)

### `SPEC-10` — Cadastro Manual Completo de Trade
- **Rastreabilidade:** `RF-M1-1` | `FN-DIA-01` | `RNF-1`, `RNF-9` | Fonte: `F7`, `F6` | Estágio 5 | Rota: `/diario`
- **História de Usuário:** Como trader, quero cadastrar ou editar manualmente uma operação completa com ativo, horários, preços, stop, alvo e resultado para manter meu diário completo mesmo sem arquivo.
- **Regras de Negócio:**
  1. Campos obrigatórios: `ticker`, `market_type` (`WIN`, `WDO`, `ACOES`, `OPCOES`), `side` (`LONG`/`SHORT`), `entry_at`, `exit_at`, `entry_price`, `exit_price`, `qty`.
  2. Campos opcionais: `stop_price`, `target_price`, `fees`, `irrf`.
  3. Resultado financeiro líquido (`net_pnl`) e percentual (`pnl_pct`) são calculados no backend de forma determinística e nunca confiados apenas ao cliente.
  4. Escrita responde em `≤ 800 ms` p95 (`RNF-1`) e dispara recálculo de agregados do dashboard.
- **Contrato de Dados (JSON):**
  ```json
  // POST /api/v1/trades -> Request
  { "ticker": "WINZ26", "side": "LONG", "entry_at": "2026-10-09T10:05:00-03:00", "exit_at": "2026-10-09T10:22:00-03:00", "entry_price": 131200, "exit_price": 131450, "qty": 2, "stop_price": 131050, "target_price": 131500, "fees": 1.80 }
  ```
- **Critérios de Aceitação:**
  - **Dado** um usuário preenchendo o modal "Novo Trade Manual" em `/diario` com dados válidos, **Quando** salvar, **Então** o trade é persistido com cálculo determinístico de resultado líquido e aparece na tabela paginada.

---

### `SPEC-11` — Anexo Seguro de Capturas de Tela de Setup
- **Rastreabilidade:** `RF-M1-2` | `FN-DIA-02` | `RNF-10` | Fonte: `F5` | Estágio 5 | Rota: `/diario`
- **História de Usuário:** Como trader, quero anexar prints do gráfico às minhas operações para revisar visualmente minha execução técnica depois do pregão.
- **Regras de Negócio:**
  1. Aceita `.png`, `.jpg`, `.jpeg`, `.webp` com limite máximo de 5 MB por imagem (até 4 imagens por trade).
  2. Armazenamento em Object Storage privado com URLs assinadas de curta duração e criptografia em repouso (`RNF-10`).
  3. Metadados EXIF são removidos no upload por segurança e privacidade.
- **Contrato de Dados (JSON):**
  ```json
  // POST /api/v1/trades/{trade_id}/attachments -> Response 201
  { "attachment_id": "uuid", "signed_url": "https://storage.../img.webp?sig=...", "mime_type": "image/webp" }
  ```
- **Critérios de Aceitação:**
  - **Dado** uma imagem PNG válida de 1,2 MB anexada na gaveta lateral do trade, **Quando** o upload concluir, **Então** a miniatura é exibida no Diário via URL assinada exclusiva do titular da conta.

---

### `SPEC-12` — Contextualizador Operacional (Setup, Plano e Motivos)
- **Rastreabilidade:** `RF-M1-3` | `FN-DIA-03` | Fonte: `F6`, `F7` | Estágio 1 e 5 | Rota: `/diario`
- **História de Usuário:** Como trader, quero registrar qual setup utilizei, se segui o plano operacional e os motivos de entrada e saída para descobrir quais estratégias realmente dão lucro.
- **Regras de Negócio:**
  1. Usuário pode selecionar setups de uma biblioteca pessoal pré-cadastrada ou criar novos inline (ex.: `Rompimento VWAP`, `Pullback M9`).
  2. Campos estruturados: `setup_id`, `followed_plan` (boolean), `entry_reason`, `exit_reason`, `notes`.
- **Contrato de Dados (JSON):**
  ```json
  // PATCH /api/v1/trades/{trade_id}/context -> Request
  { "setup_name": "Rompimento VWAP", "followed_plan": true, "entry_reason": "Volume acima da média", "exit_reason": "Alvo parcial atingido", "notes": "Boa leitura de fluxo" }
  ```
- **Critérios de Aceitação:**
  - **Dado** um trade aberto na gaveta lateral de `/diario`, **Quando** o usuário marcar `followed_plan = false` e selecionar o setup, **Então** os metadados são salvos e ficam imediatamente disponíveis para segmentação no `/analytics`.

---

### `SPEC-13` — Registro Psico-Emocional Pré e Pós-Operação
- **Rastreabilidade:** `RF-M1-4` | `FN-DIA-04` | `RNF-13` | Fonte: `F6` | Estágio 5 | Rota: `/diario`
- **História de Usuário:** Como trader, quero registrar meu estado emocional antes e depois de cada trade para identificar quando a ansiedade ou euforia prejudicam meus resultados.
- **Regras de Negócio:**
  1. Escala padronizada de 1 a 5 para estado pré-operação (`confidence`, `anxiety`) e tags pós-operação (`disciplined`, `fomo`, `revenge`, `frustrated`, `euphoric`).
  2. Dados emocionais são classificados como sensíveis pela LGPD (`RNF-13`) e protegidos pelo mesmo isolamento RLS.
- **Contrato de Dados (JSON):**
  ```json
  // PATCH /api/v1/trades/{trade_id}/psychology -> Request
  { "pre_confidence": 4, "pre_anxiety": 2, "post_tags": ["disciplined"], "emotional_note": "Tranquilo após stop" }
  ```
- **Critérios de Aceitação:**
  - **Dado** a aba emocional na gaveta do trade, **Quando** o usuário registrar `pre_anxiety = 5` e tag `revenge`, **Então** o vetor comportamental é persistido e associado ao trade para análise.

---

## 4. Módulo Trading Analytics (`CAP-03`)

### `SPEC-14` — Motor Determinístico de Métricas Core
- **Rastreabilidade:** `RF-M2-1` | `FN-ANA-01`, `FN-ANA-02`, `FN-ANA-03` | `RNF-19`, `RNF-20`, `RNF-29` | Fonte: `F6`, `F7` | Estágio 6 | Rota: `/analytics`
- **História de Usuário:** Como trader, quero ver minhas métricas reais (taxa de acerto, payoff, fator de lucro, expectativa matemática, risco/retorno médios, sequências e drawdown) calculadas com precisão matemática para avaliar minha estratégia.
- **Regras de Negócio:**
  1. Todas as métricas são calculadas exclusivamente por funções determinísticas em código (`RNF-29`), nunca por LLM:
     - `win_rate` = `(trades_vencedores / total_trades) * 100`
     - `payoff` = `ganho_medio / abs(perda_media)`
     - `profit_factor` = `lucro_bruto / abs(prejuizo_bruto)` (tratar divisão por zero retornando `null` + flag `infinite`)
     - `expectancy` = `(win_rate * ganho_medio) - ((1 - win_rate) * abs(perda_media))`
     - `max_drawdown` = maior queda pico-a-vale na série cronológica acumulada (R$ e %)
     - `max_win_streak` e `max_loss_streak` = maiores sequências consecutivas.
  2. Cobertura de testes automatizados `≥ 90%` com *golden tests* (`RNF-19`, `RNF-20`).
- **Contrato de Dados (JSON):**
  ```json
  // GET /api/v1/analytics/summary?from=2026-10-01&to=2026-10-31 -> Response 200
  { "total_trades": 40, "win_rate": 0.55, "payoff": 1.62, "profit_factor": 1.98, "expectancy_brl": 85.40, "avg_win_brl": 280.00, "avg_loss_brl": -172.84, "max_win_streak": 6, "max_loss_streak": 3, "max_drawdown_brl": -950.00 }
  ```
- **Critérios de Aceitação:**
  - **Dado** um conjunto conhecido de 10 trades (6 ganhos de R$ 200 e 4 perdas de R$ 100), **Quando** o motor calcular o resumo, **Então** retorna `win_rate = 0.60`, `payoff = 2.00`, `profit_factor = 3.00` e `expectancy_brl = 80.00`.

---

### `SPEC-15` — Gerador de Curva de Capital (Equity Curve)
- **Rastreabilidade:** `RF-M2-2` | `FN-ANA-04` | `RNF-2` | Fonte: `F5` | Estágio 6 | Rota: `/dashboard`, `/analytics`
- **História de Usuário:** Como trader, quero visualizar minha curva de capital acumulada ao longo do tempo para enxergar a consistência ou degradação da minha conta.
- **Regras de Negócio:**
  1. Ordena trades cronologicamente por `exit_at` e acumula `net_pnl` ponto a ponto (com opção de agregação diária para séries `> 500` pontos mantendo `≤ 2 s` p95 com 10.000 trades, `RNF-2`).
- **Contrato de Dados (JSON):**
  ```json
  // GET /api/v1/analytics/equity-curve -> Response 200
  { "points": [{ "timestamp": "2026-10-01T17:00:00-03:00", "cumulative_pnl": 420.50, "drawdown": 0.00 }] }
  ```
- **Critérios de Aceitação:**
  - **Dado** 10.000 operações no histórico do usuário, **Quando** solicitar a curva de capital no período, **Então** a resposta é entregue em `≤ 2 s` (p95) com série ordenada e drawdown por ponto.

---

### `SPEC-16` — Segmentador Multidimensional (Horário, Dia, Ativo, Setup)
- **Rastreabilidade:** `RF-M2-3` | `FN-ANA-05` | `RNF-1` | Fonte: `F6` | Estágio 6 | Rota: `/analytics`
- **História de Usuário:** Como trader, quero quebrar meu desempenho por faixa de horário, dia da semana, ativo e setup para descobrir onde ganho e onde perco dinheiro.
- **Regras de Negócio:**
  1. Dimensões suportadas: `hour_bucket` (hora em hora BRT), `day_of_week` (Seg–Sex), `ticker`, `setup_id`.
  2. Cada grupo retorna `trades_count`, `win_rate`, `net_pnl`, `profit_factor`.
- **Contrato de Dados (JSON):**
  ```json
  // GET /api/v1/analytics/breakdown?dimension=day_of_week -> Response 200
  { "dimension": "day_of_week", "buckets": [{ "key": "FRIDAY", "trades": 12, "win_rate": 0.25, "net_pnl": -1200.00 }] }
  ```
- **Critérios de Aceitação:**
  - **Dado** um histórico com trades em diferentes dias da semana, **Quando** selecionar a aba "Dias" em `/analytics`, **Então** o sistema agrupa corretamente os resultados por dia em `≤ 400 ms` p95.

---

### `SPEC-17` — Sintetizador de Insights Automáticos Determinísticos
- **Rastreabilidade:** `RF-M2-4` | `FN-ANA-06` | `RNF-28`, `RNF-29` | Fonte: `F6` | Estágio 6 | Rota: `/analytics`
- **História de Usuário:** Como trader, quero receber diagnósticos objetivos baseados exclusivamente nos meus números (ex.: horário ou dia com maior concentração de perdas) sem achismos ou recomendações de ativos.
- **Regras de Negócio:**
  1. Motor baseado em regras heurísticas estritas (sem LLM e sem recomendação de compra/venda, `RNF-28`).
  2. Exige amostra mínima (ex.: `≥ 5` trades no bucket) para evitar falsos positivos.
  3. Toda sentença inclui os números exatos que a originaram (rastreável, `RNF-29`).
- **Contrato de Dados (JSON):**
  ```json
  { "insights": [{ "code": "LOSS_CONCENTRATION_DAY", "severity": "warning", "message": "Sexta-feira responde por 62% das suas perdas líquidas no período (-R$ 1.200,00 em 12 operações).", "sample_size": 12 }] }
  ```
- **Critérios de Aceitação:**
  - **Dado** um período em que sexta-feira concentra `> 50%` das perdas líquidas com `≥ 5` trades, **Quando** o sintetizador rodar, **Então** retorna o insight com o percentual e valor exatos sem sugerir ativos.

---

## 5. Módulo Dashboard & Transparência (`CAP-06` / `CAP-03`)

### `SPEC-18` — Dashboard Executivo e Calendário Operacional Dinâmico
- **Rastreabilidade:** `RF-M8-1` | `FN-DASH-01`, `FN-DASH-02` | `RNF-2`, `RNF-23`, `RNF-24` | Fonte: `F5`, `F6` | Estágio 6 | Rota: `/dashboard`
- **História de Usuário:** Como trader, quero uma visão executiva imediata com cartões de KPIs, curva de capital e calendário colorido por lucro/prejuízo para avaliar minha saúde operacional em segundos.
- **Regras de Negócio:**
  1. Identidade semântica de cores (`TP2 p. 16`): Verde exclusivamente para lucro/positivo/conformidade; Vermelho para prejuízo/risco; Âmbar para alertas fiscais pendentes.
  2. Interface em PT-BR, responsiva, com suporte a modo escuro (`RNF-23`) e acessibilidade WCAG 2.1 AA (`RNF-24`).
  3. Carregamento completo em `≤ 2 s` (p95) mesmo com 10.000 operações (`RNF-2`).
- **Critérios de Aceitação:**
  - **Dado** um usuário autenticado acessando `/dashboard`, **Quando** a página carregar, **Então** exibe os cartões de KPI, a curva de capital e a grade do calendário com subtotais diários em `≤ 2 s` p95.

---

### `SPEC-19` — Gerenciador de Filtro Temporal Global
- **Rastreabilidade:** `RF-M8-2` | `FN-DASH-03` | Fonte: `F6` | Estágio 6 | Rota: `/dashboard`, `/analytics`
- **História de Usuário:** Como trader, quero selecionar um período global (`Hoje`, `7D`, `Mês Atual`, `Mês Anterior`, `Personalizado`) na barra superior e ver todos os gráficos e cartões atualizados juntos.
- **Regras de Negócio:**
  1. O estado do intervalo (`from`, `to`, `preset`) é sincronizado na URL (`query params`) e propagado a todos os widgets da tela.
- **Critérios de Aceitação:**
  - **Dado** o Dashboard exibindo "Mês Atual", **Quando** o usuário alterar o seletor global para "Mês Anterior", **Então** todos os KPIs, a curva de capital e o calendário são atualizados com o mesmo intervalo.

---

### `SPEC-20` — Componente Universal "Raio-X da Fórmula"
- **Rastreabilidade:** `RNF-29` | `FN-PLT-07` | `ADR-02` | Fonte: `F8`, `F6` | Lacuna 3 (TP2) | Rota: Global (toda métrica)
- **História de Usuário:** Como trader cético, quero clicar em um ícone `(?)` ao lado de qualquer métrica para ver a fórmula exata e os valores brutos que geraram aquele número, garantindo confiança total na plataforma.
- **Regras de Negócio:**
  1. Todo endpoint de métrica retorna ou expõe metadados de composição (`numerator`, `denominator`, `formula_latex_or_text`, `inputs_summary`).
  2. Ao clicar em `(?)`, abre gaveta/modal lateral exibindo a expressão algébrica, os valores brutos do usuário e a substituição passo a passo.
- **Contrato de Dados (JSON):**
  ```json
  // GET /api/v1/analytics/xray?metric=profit_factor&from=2026-10-01&to=2026-10-31
  { "metric": "profit_factor", "formula": "Lucro Bruto / |Prejuízo Bruto|", "inputs": { "gross_profit": 4500.00, "gross_loss": -2272.73 }, "result": 1.98 }
  ```
- **Critérios de Aceitação:**
  - **Dado** o cartão de "Fator de Lucro" no `/dashboard`, **Quando** o usuário clicar no ícone `(?)`, **Então** a gaveta Raio-X exibe a fórmula, o Lucro Bruto, o Prejuízo Bruto e a divisão exata que resulta no valor exibido.

---

## 6. Módulo Gestão de Risco (`CAP-05`)

### `SPEC-21` — Gestão de Parâmetros e Tetos de Risco
- **Rastreabilidade:** `RF-M7-1` | `FN-RISC-01` | Fonte: `F6`, `F7` | Estágio 2 | Rota: `/risco/parametros`
- **História de Usuário:** Como trader, quero cadastrar meus limites máximos de perda por operação, dia, semana e mês para não depender apenas da memória durante o pregão.
- **Regras de Negócio:**
  1. Aceita limites em valor monetário (R$) e/ou percentual do capital de referência.
  2. Validação: `max_loss_per_trade <= max_loss_daily <= max_loss_weekly <= max_loss_monthly`.
- **Contrato de Dados (JSON):**
  ```json
  // PUT /api/v1/risk/parameters -> Request
  { "capital_brl": 20000.00, "max_loss_trade_brl": 200.00, "max_loss_day_brl": 600.00, "max_loss_week_brl": 1800.00, "max_loss_month_brl": 4000.00 }
  ```
- **Critérios de Aceitação:**
  - **Dado** um usuário tentando salvar `max_loss_day_brl` menor que `max_loss_trade_brl`, **Quando** enviar o formulário, **Então** o sistema rejeita com erro de validação claro indicando a inconsistência hierárquica.

---

### `SPEC-22` — Calculadora Passiva de Tamanho de Posição com Atalho Global (`Alt+C`/`Ctrl+K`)
- **Rastreabilidade:** `RF-M7-2` | `FN-RISC-02` | `RNF-28` | Fonte: `F6`, `F7` | Retorno Avaliador TP2 | Estágio 4 | Rota: `/risco/calculadora` + Drawer Global
- **História de Usuário:** Como trader prestes a entrar em uma operação, quero abrir rapidamente a calculadora por atalho de teclado em qualquer tela para saber quantos contratos ou ações posso operar sem ultrapassar meu limite de perda.
- **Regras de Negócio:**
  1. Ferramenta estritamente **passiva e determinística** (`RNF-28`): calcula lote a partir dos parâmetros do usuário (`entry_price`, `stop_price`, `target_price`, `risk_brl`), sem botão de "Comprar" e sem sugerir ativos ou direções.
  2. Cálculo por tipo de ativo:
     - `WIN`: R$ 0,20 por ponto por contrato (`contratos = floor(risk_brl / (abs(entry - stop) * 0.20))`)
     - `WDO`: R$ 10,00 por ponto por contrato (`contratos = floor(risk_brl / (abs(entry - stop) * 10.00))`)
     - `ACOES`: `lotes = floor(risk_brl / abs(entry - stop))`
  3. Se o risco informado exceder o teto diário restante (`max_loss_day - perda_do_dia`), exibe alerta visual vermelho sem bloquear o uso da calculadora.
  4. Acessível via rota `/risco/calculadora` e via atalho global `Alt+C` (ou `Ctrl+K` → Calculadora) como drawer flutuante.
- **Critérios de Aceitação:**
  - **Dado** um trader em `/diario` com risco por trade de R$ 200,00, **Quando** pressionar `Alt+C` e informar entrada `131000` e stop `130800` em `WIN`, **Então** o drawer exibe `5 contratos` (200 pts × R$ 0,20 = R$ 40/contrato) sem navegar para fora da tela atual.

---

## 7. Módulo Gestão Fiscal B3 (`CAP-04`)

### `SPEC-23` — Segregador Tributário (Day Trade vs Swing Trade) e Isenção R$ 20k
- **Rastreabilidade:** `RF-M5-1` | `FN-FISC-01`, `FN-FISC-02` | `RNF-20` | Fonte: `F4` | Estágio 7 | Rota: `/fiscal`
- **História de Usuário:** Como investidor B3, quero que minhas operações sejam automaticamente separadas entre Day Trade e Swing Trade e que a isenção mensal de R$ 20.000 em vendas de ações seja verificada corretamente.
- **Regras de Negócio:**
  1. Classificação automática: operações abertas e encerradas no mesmo pregão no mesmo ativo/CPF vão para o silo `DAY_TRADE` (alíquota 20%, **sem isenção**). Operações encerradas em pregão posterior vão para `SWING_TRADE` (alíquota 15%).
  2. Verificação da isenção de R$ 20.000 (`F4`): soma o **volume bruto de alienações (vendas)** de **ações** em Swing Trade no mês civil. Se `<= R$ 20.000,00`, o lucro líquido de Swing Trade em ações é marcado como **Isento e Não Tributável** (não consome prejuízo acumulado nem gera imposto). Se `> R$ 20.000,00`, o lucro é integralmente tributável.
- **Critérios de Aceitação:**
  - **Dado** um mês com R$ 18.500,00 em vendas de ações em Swing Trade (lucro de R$ 2.000,00) e lucro de R$ 500,00 em Day Trade, **Quando** a apuração mensal rodar, **Então** Swing Trade é marcado como isento (imposto R$ 0,00) e Day Trade é tributado em 20% (R$ 100,00).

---

### `SPEC-24` — Saldo Inicial de Prejuízos Anteriores (`/fiscal/ajustes`)
- **Rastreabilidade:** `RF-M5-2` | `FN-FISC-03` | `RNF-30` | Fonte: `F4` | Lacuna 2 (TP2) | Estágio 7 | Rota: `/fiscal/ajustes`
- **História de Usuário:** Como novo usuário que já operava antes de usar o TradePilot AI, quero informar meu saldo inicial de prejuízos acumulados por modalidade para que o sistema os abata antes de calcular minha primeira DARF.
- **Regras de Negócio:**
  1. Permite informar saldo negativo inicial separado por silo (`DAY_TRADE`, `SWING_TRADE`) com competência de referência (`YYYY-MM`).
  2. Toda criação ou alteração gera registro imutável em `audit_logs` (`SPEC-04`).
- **Contrato de Dados (JSON):**
  ```json
  // PUT /api/v1/fiscal/prior-losses -> Request
  { "reference_month": "2026-09", "day_trade_loss_brl": 3200.00, "swing_trade_loss_brl": 850.00, "reason": "Saldo trazido da declaração anterior" }
  ```
- **Critérios de Aceitação:**
  - **Dado** um usuário cadastrando R$ 3.200,00 de prejuízo anterior em Day Trade em `/fiscal/ajustes`, **Quando** apurar outubro com lucro de R$ 2.000,00 em Day Trade, **Então** o lucro é totalmente compensado e o saldo restante de prejuízo passa a ser R$ 1.200,00.

---

### `SPEC-25` — Compensador de Prejuízos por Silo e Apurador de Imposto / IRRF
- **Rastreabilidade:** `RF-M5-1`, `RF-M5-2` | `FN-FISC-03`, `FN-FISC-04` | `RNF-19`, `RNF-20` | Fonte: `F4` | Estágio 7 | Rota: `/fiscal`
- **História de Usuário:** Como trader, quero que o sistema compense prejuízos acumulados estritamente dentro da mesma modalidade e deduza o IRRF já retido na nota de corretagem.
- **Regras de Negócio:**
  1. **Regra de Silo Estrita:** Prejuízo de Day Trade **nunca** compensa lucro de Swing Trade e vice-versa (garantido em código e schema).
  2. Custos operacionais (taxas B3 e corretagem) são deduzidos do resultado bruto antes da base de cálculo.
  3. Sobre a base líquida positiva após compensação, aplica-se `20%` (Day Trade) ou `15%` (Swing Trade) e deduz-se o IRRF retido em nota (`1%` DT / `0,005%` ST).
  4. Cobertura de testes `≥ 90%` com *golden tests* (`RNF-19`, `RNF-20`).
- **Critérios de Aceitação:**
  - **Dado** prejuízo acumulado de R$ 1.000,00 em Swing Trade e lucro de R$ 1.000,00 em Day Trade no mês, **Quando** a apuração fechar, **Então** o prejuízo de Swing não abate o Day Trade e o imposto de Day Trade é calculado integralmente sobre R$ 1.000,00 (R$ 200,00 menos IRRF retido).

---

### `SPEC-26` — Gerador de DARF Código 6015 (Valor ≥ R$ 10,00 e Vencimento)
- **Rastreabilidade:** `RF-M5-3` | `FN-FISC-05` | `RNF-7` | Fonte: `F4` | Estágio 7 | Rota: `/fiscal/darf`
- **História de Usuário:** Como trader com imposto devido no mês, quero gerar os dados exatos da DARF código 6015 (competência, valor e vencimento no último dia útil do mês seguinte) para pagar sem atraso e sem multa.
- **Regras de Negócio:**
  1. Consolida imposto líquido devido dos silos no mês (`SPEC-25`).
  2. Se `total_tax_due < R$ 10,00` (regra da Receita Federal), **não gera DARF** e acumula o valor para a competência subsequente (`carried_over_tax`).
  3. Se `>= R$ 10,00`, gera documento estruturado com Código da Receita `6015`, período de apuração, valor principal e data de vencimento calculada considerando o calendário de feriados nacionais/bancários (último dia útil do mês subsequente).
  4. Permite ao usuário marcar manualmente como "DARF Paga / Agendada" para controle de status.
- **Critérios de Aceitação:**
  - **Dado** um imposto líquido apurado de R$ 8,50 em outubro, **Quando** o usuário abrir `/fiscal/darf`, **Então** o sistema informa que o valor é inferior a R$ 10,00 e acumula R$ 8,50 para novembro.
  - **Dado** um imposto líquido apurado de R$ 145,00 em outubro, **Quando** gerar a DARF, **Então** exibe código `6015`, valor `R$ 145,00` e vencimento no último dia útil de novembro.

---

### `SPEC-27` — Emissor de Relatório Mensal e Memória de Cálculo Auditável em PDF
- **Rastreabilidade:** `RF-M5-4` | `FN-FISC-06` | `RNF-29` | Fonte: `F4` | Estágio 7 | Rota: `/fiscal/darf`
- **História de Usuário:** Como trader ou contador, quero baixar um relatório mensal em PDF com a memória de cálculo completa (operações, taxas, compensações e alíquotas) para arquivamento e conferência contábil.
- **Regras de Negócio:**
  1. O PDF lista por silo: operações encerradas no mês, total de vendas (para verificação de isenção), resultado bruto, taxas deduzidas, prejuízo anterior compensado, base de cálculo, alíquota aplicada, IRRF deduzido e imposto líquido/DARF.
  2. Inclui aviso regulatório explícito de que o relatório auxilia a apuração e não substitui a responsabilidade profissional do contador (Escopo Negativo).
- **Critérios de Aceitação:**
  - **Dado** um fechamento fiscal mensal concluído, **Quando** o usuário clicar em "Baixar Memória de Cálculo Auditável em PDF", **Então** o sistema gera e faz download de um PDF contendo todas as etapas matemáticas e a lista de operações da competência.
