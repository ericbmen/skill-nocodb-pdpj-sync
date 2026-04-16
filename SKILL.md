---
name: nocodb-pdpj-sync
description: Skill de automação para ler CPFs novos no NocoDB, pesquisar no PDPJ e atualizar as observações automaticamente.
skills:
  - pdpj-researcher
---

# NocoDB + PDPJ Sync Skill

## 🎯 Objetivo
Varredura autônoma em tabelas do NocoDB para realizar background-check (Criminal/Trabalhista) via PDPJ/CNJ de forma sequencial e segura.

## 🧠 Fluxo de Execução (REGRAS DE OURO)

1. **BUSCA DE PENDÊNCIAS (NocoDB)**
   - Ferramenta: `mcp_nocodb-pesquisas-empresas_queryRecords`.
   - Tabela: `mxr0nh36qllj0w2`.
   - Filtro: `(CPF,notblank)~and(OBS. CRIMINAIS,blank)`.
   - **IMPORTANTE**: Processar um registro por vez (limit: 1) para evitar timeouts e garantir integridade.

2. **EXTRAÇÃO E LEITURA (Deep Dive):**
   - Para **cada** processo que passou no filtro, utilize a ferramenta `pdpj_visao_geral_processo` para capturar as informações estruturais (Tribunal, Partes, Status, Valor).
   - Acione imediatamente a ferramenta `pdpj_analise_essencial`. Você **deve** usar esta ferramenta para acessar os documentos iniciais e últimas decisões.
   - **OBRIGATÓRIO:** Identificar a capitulação penal (Artigo e Crime). Se for Execução Penal, buscar o processo originário para confirmar o crime cometido.

3. **SÍNTESE DO ANALISTA (Módulo IA):**
   - Com base nos documentos lidos, escreva um **Resumo Executivo de Inteligência**.
   - O resumo deve ser focado no modus operandi, na infração penal (especificando o crime) ou no motivo grave da lide disfarçada sob jargão jurídico. Vá direto ao fato ilícito/grave.

4. **FORMATAÇÃO DE RESULTADOS (PADRÃO OBRIGATÓRIO)**
   - **NADA**: Se não houver processos graves, escrever `"NADA"`.
   - **CPF INVÁLIDO**: Se o CPF for inválido ou parcial, escrever `"NADA (CPF PARCIAL/FORMATO INVÁLIDO)"`.
   
   - **CASOS POSITIVOS (MULTI-PROCESSO)**:
     - Se houver mais de um processo, cada um deve ser inserido separadamente com **UMA LINHA EM BRANCO**, seguida por **TRÊS HÍFENS (---)** e **OUTRA LINHA EM BRANCO** como divisor visual obrigatório.
      
     - **TEMPLATE CRIMINAL**:
       PROCESSO CRIMINAL Nº [NUMERO]
       CRIME/CAPITULAÇÃO: [ARTIGO E NOME DO CRIME - EX: ART. 33 - TRÁFICO DE DROGAS]
       AUTOR: [NOME DA PARTE ATIVA]
       [RÉU/ACUSADO/APENADO/INVESTIGADO]: [NOME DA PARTE PASSIVA]
       ORIGEM/JUSTIÇA: [VARA/COMARCA] - [TRIBUNAL]
       INSTÂNCIA: [GRAU DE JURISDIÇÃO]
       ANO DO PROCESSO: [ANO]
       STATUS: [STATUS]
       RESUMO: [RESUMO INTELIGENTE OBRIGATORIAMENTE INCLUINDO O FATO GERADOR E CAPITULAÇÃO PENAL]

     - **TEMPLATE TRABALHISTA**:
       PROCESSO TRABALHISTA Nº [NUMERO]
       RECLAMANTE: [NOME DA PARTE ATIVA]
       RECLAMADO: [NOME DA PARTE PASSIVA]
       ORIGEM/JUSTIÇA: [VARA/COMARCA] - [TRIBUNAL]
       ANO DO PROCESSO: [ANO]
       VALOR DA CAUSA: [VALOR]
       STATUS: [STATUS]
       ASSUNTO TÉCNICO: [ASSUNTO]
       RESUMO: [RESUMO INTELIGENTE FOCADO NO MOTIVO DA LIDE E RISCO EMPRESARIAL]

5. **ATUALIZAÇÃO (NocoDB)**
   - Ferramenta: `mcp_nocodb-pesquisas-empresas_updateRecords`.
   - Coluna Criminal (ID): `c7on89qkiyfw9k7`.
   - Coluna Trabalhista (ID): `cdx78fvc5ekm0pp`.

5. **RELATÓRIO FINAL**
   - Ao final do lote, informar ao usuário o total de registros sincronizados com sucesso.

## 🛠️ Comandos de Gatilho
- "Sincronizar CPFs pendentes"
- "Executar automação NocoDB PDPJ"
- "Auditores pendentes no NocoDB"
