# NocoDB + PDPJ Sync Skill

Repositório da skill de automação para realizar background checks jurídicos (Criminais e Trabalhistas) de forma automatizada.

## 🚀 Como instalar

1. Clone este repositório na sua pasta de skills:
   ```bash
   git clone https://github.com/ericbmen/skill-nocodb-pdpj-sync.git .agent/skills/nocodb-pdpj-sync
   ```

2. Configure o arquivo `.env` baseado no `.env.example`.

## ⚙️ Configurações Necessárias

O arquivo `.env` deve conter:
- `NOCODB_API_KEY`: Sua chave de API do NocoDB.
- `TECJUSTICA_TOKEN`: Seu token de acesso ao servidor PDPJ.
- `NOCODB_TABLE_ID`: O ID da tabela alvo no NocoDB.

## 🤖 Uso
Basta pedir ao assistente: *"Sincronizar CPFs pendentes na tabela do NocoDB"* ou *"Executar automação PDPJ"*.
