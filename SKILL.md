---
name: nocodb-pdpj-sync
description: Skill de automação para ler CPFs novos no NocoDB, pesquisar no PDPJ usando a técnica do pdpj-researcher, e atualizar as observações Criminais e Trabalhistas no NocoDB automaticamente.
---

# NocoDB + PDPJ Sync Skill

> Agente de sincronização autônoma: Conecta a tabela do NocoDB aos robôs do servidor MCP TecJustiça para preencher fichas de inteligência automaticamente.

## 🎯 Objetivo Global

> **Orientação MÁXIMA:** A partir de agora, é OBRIGATÓRIO que ao atualizar os campos **OBS. CRIMINAIS** e **OBS. TRABALHISTAS**, você utilize resumos altamente detalhados. O campo "RESUMO" não deve ser genérico; ele deve conter uma explicação técnica e fatídica de **3 a 5 linhas** extraída obrigatoriamente via `pdpj_analise_essencial`.

Realizar varreduras periódicas ou sob demanda nas tabelas do NocoDB em busca de registros (funcionários/motoristas) que ainda não foram auditados criminalmente e trabalhisticamente. Pesquisá-los no PDPJ (CNJ) e salvar os resultados de volta na plataforma de banco de dados, fechando o ciclo de background-check.

## 🧠 Fluxo de Execução Obrigatório

Sempre que o usuário solicitar a sincronização (ex: "Sincronizar NocoDB", "Processar CPFs pendentes", "Rode o sync"), execute de forma autônoma os seguintes passos:

### 1. BUSCA DE PENDÊNCIAS (NocoDB)
Utilize a ferramenta `mcp_nocodb_pesquisas_empresas_queryRecords` especificando a `tableId` desejada (ex: `mxr0nh36qllj0w2` para Base_empresas).
- **Filtro obrigatório:** Onde o CPF existe E a coluna "OBS. CRIMINAIS" esteja vazia. Em syntax NocoDB OData: `(CPF,notblank)~and(OBS. CRIMINAIS,blank)` ou use os IDs dos campos.

### 1.5. VALIDAÇÃO DE CPF (antes de consultar o PDPJ)
Antes de chamar o PDPJ, valide o CPF usando o algoritmo oficial de dígitos verificadores:

```
function validarCPF(cpf) {
  cpf = cpf.replace(/\D/g, '');
  if (cpf.length !== 11) return false;
  // Rejeita sequências de dígitos iguais (111.111.111-11, etc.)
  if (/^(\d)\1{10}$/.test(cpf)) return false;

  // Primeiro dígito verificador
  let soma = 0;
  for (let i = 0; i < 9; i++) soma += parseInt(cpf[i]) * (10 - i);
  let resto = soma % 11;
  let dv1 = resto < 2 ? 0 : 11 - resto;
  if (parseInt(cpf[9]) !== dv1) return false;

  // Segundo dígito verificador
  soma = 0;
  for (let i = 0; i < 10; i++) soma += parseInt(cpf[i]) * (11 - i);
  resto = soma % 11;
  let dv2 = resto < 2 ? 0 : 11 - resto;
  return parseInt(cpf[10]) === dv2;
}
```

**Se o CPF for inválido:** preencha ambas as colunas com `"NADA (CPF INVÁLIDO — DÍGITOS VERIFICADORES)"` e pule para o próximo registro.

### 2. AUDITORIA NO PDPJ (TecJustiça)
Para cada registro com CPF válido:
- Utilize a ferramenta `mcp_tecjustica_pdpj_buscar_processos` buscando pelo CPF da pessoa.
- **RESTRIÇÃO DE ESFERA (CRÍTICO):** Ignore sumariamente qualquer processo que NÃO seja das esferas **CRIMINAL/PENAL** ou **TRABALHISTA**.
- **PROIBIÇÃO:** É terminantemente proibido inserir processos de Execução de Título Extrajudicial, Cobranças Cíveis, Inventários ou qualquer outra lide estritamente cível na tabela. Se o CPF possuir apenas processos cíveis, o resultado final deve ser `"NADA"`.
- Se houver processos Criminais ou Trabalhistas, use **OBRIGATORIAMENTE** a ferramenta `pdpj_analise_essencial` para extrair o resumo em 3-5 linhas.

### 2.5. REGRA CRÍTICA: INTERPRETAÇÃO DAS PARTES NO PDPJ
⚠️ **NUNCA copie cegamente os dados de partes do PDPJ.** Use lógica jurídica:
- **Processos Trabalhistas:** A pessoa pesquisada (CPF) é quase sempre o **RECLAMANTE**. A empresa é o **RECLAMADO**.
- **Processos Criminais:** A pessoa pesquisada (CPF) é tipicamente o **RÉU**. O **AUTOR** é o Ministério Público ou vítima.

### 3. FORMATAÇÃO E SEPARAÇÃO DOS DADOS
- Crie strings separadas para as duas colunas seguindo o **PADRÃO IMUTÁVEL** especificado abaixo. 
- **CABEÇALHOS:** Você deve OBRIGATORIAMENTE iniciar a string com o título Markdown `## INFORMAÇÕES CRIMINAIS` ou `## INFORMAÇÕES TRABALHISTAS`.
- **MULTIPLICIDADE:** Se um CPF possuir mais de um processo relevante, separe-os com a linha de 142 traços:
  `--------------------------------------------------------------------------------------------------------------------------------------------------------------`

**A) OBS. CRIMINAIS (campo `c7on89qkiyfw9k7`):**
```markdown
## INFORMAÇÕES CRIMINAIS

PROCESSO CRIMINAL Nº [número]
AUTORIDADE POLICIAL: [delegacia ou comando, se disponível]
RÉU: [nome completo]
ORIGEM/JUSTIÇA: [comarca/vara e tribunal]
INSTÂNCIA: [grau - ex: 1º GRAU]
ANO DO PROCESSO: [ano]
VALOR DA CAUSA: NÃO APLICÁVEL (processo criminal)
STATUS: [situação atual detalhada entre parênteses]
ASSUNTO TÉCNICO: [tipificação exata do crime e artigo, se houver]
RESUMO: [Texto robusto e detalhado extraído do pdpj_analise_essencial]
```

**B) OBS. TRABALHISTAS (campo `cdx78fvc5ekm0pp`):**
```markdown
## INFORMAÇÕES TRABALHISTAS

PROCESSO TRABALHISTA Nº [número]
RECLAMANTE: [nome completo ou razão social]
RECLAMADO: [nome completo ou razão social]
ORIGEM/JUSTIÇA: [vara do trabalho e tribunal]
INSTÂNCIA: [grau - ex: 1º GRAU]
ANO DO PROCESSO: [ano]
VALOR DA CAUSA: [valor exato e observação se é valor de acordo]
STATUS: [situação atual detalhada entre parênteses]
ASSUNTO TÉCNICO: [assunto principal da lide]
RESUMO: [Texto robusto e detalhado extraído do pdpj_analise_essencial]
```

- Se não houver ocorrências relevantes após a triagem, insira apenas a string `"NADA"`.
- Se o PDPJ retornar erro (404, 500, etc.) mesmo com CPF válido, insira `"NADA"`.

### 4. REGRA DE IMUTABILIDADE (DITADURA DO PADRÃO)
⚠️ **ESTE PADRÃO DE FORMATAÇÃO É IMUTÁVEL.** 
1. Jamais modifique a ordem dos campos ou os rótulos.
2. **NUNCA** omita um campo do gabarito (se não houver o dado, preencha com "Não informado").
3. **QUEBRAS DE LINHA REAIS:** Ao chamar a ferramenta `updateRecords`, certifique-se de enviar quebras de linha reais na string do JSON, não o caractere literal `\n`. O usuário deve ver o texto em blocos, não em uma linha única suja.

### 5. CHECKLIST DE PRÉ-ENVIO (MANDATÓRIO)
Antes de executar o `updateRecords`, verifique:
- [ ] O título `## INFORMAÇÕES ...` está no topo?
- [ ] O campo `VALOR DA CAUSA:` está presente (mesmo em Criminais como "N/A")?
- [ ] O `RESUMO:` tem no mínimo 3 linhas de conteúdo real?
- [ ] Enviei quebras de linha reais no JSON?
- [ ] Removi lites cíveis/cobranças de banco?

### 6. ATUALIZAÇÃO EM MASSA (NocoDB)
Utilize a ferramenta `mcp_nocodb_pesquisas_empresas_updateRecords` enviando o array de resultados processados. No máximo 10 registros por chamada.

... (mantém o resto do arquivo)