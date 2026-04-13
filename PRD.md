### 1. Visão Geral do Projeto
O **CreAuto** é um orquestrador de **Processamento Inteligente de Documentos (IDP)** e **IA Agêntica**. O objetivo central é eliminar a configuração manual de templates (o "cansaço da configuração") utilizando **Modelos de Linguagem Visual (VLMs)** para interpretar documentos de forma semântica e agnóstica ao layout. O sistema automatiza o fluxo entre Planilhas Google e sistemas legados como **e2doc** e o portal **SIC (Confea)**.

### 2. Diferencial Estratégico
Diferente de OCRs tradicionais que falham em layouts variados, o CreAuto utiliza a lógica de **Cadeia de Pensamento (Chain of Thought - CoT)** para "raciocinar" sobre a estrutura visual (ex: identificar que um "O" em uma coluna de preços é, na verdade, um "0").

### 3. Funcionalidades Essenciais (MVP)
*   **Extração Zero-Template:** Leitura de documentos sem necessidade de treinar modelos ou desenhar caixas delimitadoras.
*   **Orquestração de Agente (RPA):** Uso de robôs para navegar, pesquisar e cadastrar dados em portais web (e2doc e SIC).
*   **Escrita de Caminho Direto:** Inserção estruturada de dados diretamente no Google Sheets, eliminando arquivos CSV intermediários.
*   **Human-in-the-Loop (HITL):** Interface de revisão para campos com pontuação de confiança abaixo de 80%.
*   **Notificações Sonoras de Estado:** Gatilhos auditivos distintos para conclusão de processos ou alertas de erro.

### 4. Jornada do Usuário e Páginas
O Antigravity deve implementar as seguintes telas seguindo este fluxo:

1.  **Tela 1: Dashboard de Login (Supabase)**
    *   Autenticação via Supabase Auth.
    *   **Lógica:** Se for o primeiro acesso, redirecionar para a Tela 2.

2.  **Tela 2: Configurações de Fluxo de Trabalho**
    *   **Campos:** Link da Planilha Google, Credenciais e2doc (Empresa/Login/Senha), Credenciais SIC (Login/Senha) e Intervalo de Execução.
    *   **Segurança:** Armazenamento criptografado (pgcrypto) no Supabase.

3.  **Tela 3: Tela Principal (Dashboard de 4 Botões)**
    *   Os botões na interface não devem possuir aspas no nome:
    *   **Botão 1: Configurações de Fluxo de Trabalho** (Editar parâmetros da Tela 2).
    *   **Botão 2: Pesquisa Doc** (Ativa pesquisa no e2doc pelo protocolo da planilha; emite **Notificação Sonora A** ao finalizar).
    *   **Botão 3: Cadastramento SIC** (Executa cadastramento em 4 etapas detalhadas abaixo; emite **Notificação Sonora B** em pausas de erro e **C** na conclusão).
    *   **Botão 4: Cadastramento InfoCrea** (Visível, mas desabilitado para versões futuras).

### 5. Fluxo Detalhado do Cadastramento SIC (Botão 3)
A automação deve seguir rigorosamente estas etapas no portal:
1.  **CADASTRAR PROFISSIONAL:** Input inicial dos dados extraídos.
2.  **CADASTRO DE DADOS PESSOAIS DO PROFISSIONAL:** Preencher campos obrigatórios (*), pulando o que já foi inserido na etapa anterior.
3.  **CADASTRO DE ENDEREÇO(S) DO PROFISSIONAL.**
4.  **CADASTRO DE TÍTULO DO PROFISSIONAL.**
*   **HITL:** Se houver erro de validação ou profissional já cadastrado, o sistema **pausa**, notifica o usuário sonoramente e aguarda o clique em "Continuar".

### 6. Stack Tecnológica Recomendada
Para o desenvolvimento no Antigravity, utilize esta arquitetura:
*   **Frontend:** Next.js (App Router) + Tailwind CSS.
*   **Backend/DB:** Supabase (PostgreSQL, Storage e Auth).
*   **Automação:** Playwright (para navegação resiliente em sistemas legados).
*   **Motor de IA:** API do **Gemini 1.5 Flash** (janela de 1M de tokens e custo de ~$1 por 6.000 páginas).
*   **Lógica de Agentes:** **Pydantic AI** para garantir saídas JSON estruturadas e seguras.
*   **Integração Sheets:** Google Sheets API v4 (focando em operações `batchUpdate` para performance superior).

### 7. Design System e UI/UX
*   **Estética:** Design limpo e corporativo (Clean & Enterprise).
*   **Revisão Side-by-Side:** Exibir o documento original ao lado dos dados extraídos com **bounding boxes** (caixas delimitadoras) destacando a origem de cada valor.
*   **Performance:** Implementar **Context Caching** no Gemini para instruções de sistema longas, reduzindo custos em até 90%.

### 8. Schema do Banco de Dados (Supabase)
*   `user_configs`: `id`, `user_id`, `sheets_url`, `e2doc_creds_enc`, `sic_creds_enc`, `interval`, `updated_at`.
*   `extractions`: `id`, `workflow_id`, `status` (pending_review/completed), `confidence_score`, `raw_json`, `document_path`.
*   `automation_logs`: `id`, `action_type` (e2doc/sic), `step` (1-4), `status`, `message`, `timestamp`.