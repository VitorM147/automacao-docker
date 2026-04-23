# GitLab Dockerfile Scanner - Inventário Docker em Massa

Script Python para escanear automaticamente todos os projetos de um grupo do GitLab (incluindo subgrupos), identificar Dockerfiles e imagens Docker em pipelines (`.gitlab-ci.yml`), classificar por ambiente e sistema operacional, e gerar relatórios em Excel.

## 📋 Pré-requisitos

- Python 3.8+
- Bibliotecas: `requests`, `openpyxl`, `python-dotenv`
- Token de acesso pessoal do GitLab com escopo `read_api`
- Permissão de leitura nos projetos do grupo
- Acesso ao grupo/subgrupos que deseja escanear

### Instalação das dependências

```bash
pip install -r requirements.txt
```

## 🏗️ Estrutura do Projeto

| Arquivo | Descrição |
|---------|-----------|
| `gitlab_dockerfile_scanner_v2.py` | Scanner principal com ThreadPool adaptativo + Checkpoint + Excel parcial |
| `gerar_excel_parcial.py` | Gera Excel a partir do checkpoint (sem esperar o scan terminar) |
| `gerar_relatorio_drift.py` | Gera relatório analítico de drift de imagens Docker |
| `check_pendentes.py` | Verifica projetos pendentes e suas branches/Dockerfiles |
| `requirements.txt` | Dependências do projeto |

## 🖥️ Uso via Linha de Comando (CLI)

### Configurar o Token

Windows (PowerShell):
```powershell
$env:GITLAB_TOKEN = "glpat-xxxxxxxxxxxxxxxxxxxx"
```

Windows (CMD):
```cmd
set GITLAB_TOKEN=glpat-xxxxxxxxxxxxxxxxxxxx
```

Linux/macOS:
```bash
export GITLAB_TOKEN="glpat-xxxxxxxxxxxxxxxxxxxx"
```

Ou crie um arquivo `.env` na raiz do projeto:
```
GITLAB_TOKEN=glpat-xxxxxxxxxxxxxxxxxxxx
```

### Executar o Scanner

```bash
# Executar o scanner completo (todas as branches de todos os projetos)
python gitlab_dockerfile_scanner_v2.py
```

### Gerar Excel Parcial (durante a execução)

```bash
# Gera Excel com os dados já coletados sem interromper o scanner
python gerar_excel_parcial.py
```

### Gerar Relatório de Drift

```bash
# Gera relatório analítico a partir do Excel ou checkpoint
python gerar_relatorio_drift.py
```

### Verificar Projetos Pendentes

```bash
# Mostra projetos ainda não processados com detalhes de branches
python check_pendentes.py
```

## ⚙️ Configuração

As configurações ficam no topo do arquivo `gitlab_dockerfile_scanner_v2.py`:

| Variável | Padrão | Descrição |
|----------|--------|-----------|
| `GITLAB_URL` | `https://gitlab.com` | URL da instância GitLab |
| `GROUP_ID` | `grupo-dpsp` | ID ou path do grupo a escanear |
| `OUTPUT_FILE` | `dockerfiles_dpsp.xlsx` | Nome do Excel final |
| `PARTIAL_FILE` | `dockerfiles_dpsp_parcial.xlsx` | Nome do Excel parcial |
| `CHECKPOINT_FILE` | `checkpoint.json` | Arquivo de checkpoint |
| `MAX_WORKERS` | `10` | Número máximo de threads paralelas |
| `CHECKPOINT_INTERVAL` | `10` | Salva checkpoint a cada N projetos |
| `EXCEL_INTERVAL` | `50` | Salva Excel parcial a cada N projetos |

### Variáveis de Ambiente

| Variável | Obrigatória | Descrição |
|----------|-------------|-----------|
| `GITLAB_TOKEN` | ✅ Sim | Token de acesso pessoal do GitLab |

## 🚀 Funcionalidades

### Scanner (v2)

- **ThreadPool adaptativo**: 10 workers paralelos com rate limiter que lê os headers `RateLimit-Remaining` do GitLab em tempo real
- **Checkpoint automático**: salva progresso a cada 10 projetos em JSON. Se interromper, retoma de onde parou
- **Excel parcial**: gera Excel intermediário a cada 50 projetos como backup
- **Varredura completa**: escaneia todas as branches de todos os projetos
- **Detecção de SO**: identifica o sistema operacional base de cada imagem Docker
- **Classificação de ambiente**: classifica branches em Prod, QA ou Dev
- **Retry com backoff**: trata erros de rede e HTTP 429 automaticamente

### Relatório de Drift

Gera Excel analítico com 7 abas:

| Aba | Conteúdo |
|-----|----------|
| Resumo Executivo | Números chave, indicadores de risco, índice de padronização |
| Por Runtime | Distribuição por tecnologia (Node.js, Java, Python, Go, etc.) |
| Top 20 Imagens | Imagens mais usadas com flags de `:latest` e EOL |
| Uso de latest | Lista completa de quem usa `:latest` (Prod destacado em vermelho) |
| Versões EOL | Todas as imagens com versão fim de vida |
| Por SO | Distribuição por sistema operacional |
| Produção - Detalhes | Top 10 imagens em Prod com alertas |

## 📊 Exemplo de Saída

```
============================================================
  Scanner de Dockerfiles - Grupo DPSP (v2)
  Modo: ThreadPool adaptativo + Checkpoint
============================================================

Buscando projetos do grupo 'grupo-dpsp'...
  Pagina 1: +100 (total: 100)
  Pagina 2: +100 (total: 200)
  ...
  -> 1365 projetos encontrados.
  -> 0 ja processados (checkpoint)
  -> 1365 pendentes

[1/1365] grupo-dpsp/projeto-a -> 5 registros
    [Prod] main | Dockerfile: node:20-slim (Debian (slim))
    [Dev] develop | Pipeline: python:3.11 (Debian (Python))
[2/1365] grupo-dpsp/projeto-b -> Nenhum Docker encontrado
  [CHECKPOINT] Salvo: 10/1365 projetos | 45 registros | API remaining: 280/300 | ETA: 65.2 min
...

============================================================
  CONCLUIDO em 72.3 minutos
  1365 projetos processados
  329 projetos com Docker
  17979 registros no total
  76869 requests a API
  0 pausas por rate limit
============================================================

Arquivo gerado: dockerfiles_dpsp.xlsx
```

## 🔐 Configuração do Token no GitLab

### Passo 1: Criar o Token de Acesso

1. Acesse **GitLab → User Settings → Access Tokens**
2. Crie um token com:
   - **Nome**: `dockerfile-scanner`
   - **Expiration**: Defina uma data ou deixe sem expiração
   - **Scopes**: `read_api` (obrigatório)
3. Copie o token gerado

### Passo 2: Configurar o Token

Crie um arquivo `.env` na raiz do projeto:
```
GITLAB_TOKEN=glpat-xxxxxxxxxxxxxxxxxxxx
```

> ⚠️ O arquivo `.env` está no `.gitignore` e nunca será enviado ao repositório.

## 🚀 Uso via Pipeline (GitLab CI/CD)

### Execução agendada

```yaml
# .gitlab-ci.yml

stages:
  - scan

variables:
  GROUP_ID: "grupo-dpsp"

scan-dockerfiles:
  stage: scan
  image: python:3.11-slim
  before_script:
    - pip install requests openpyxl python-dotenv
  script:
    - python gitlab_dockerfile_scanner_v2.py
  artifacts:
    paths:
      - dockerfiles_dpsp.xlsx
      - dockerfiles_dpsp_parcial.xlsx
    expire_in: 30 days
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
    - if: $CI_PIPELINE_SOURCE == "web"
      when: manual
```

### Configurar Token no CI/CD

1. Vá em **Settings → CI/CD → Variables**
2. Adicione:
   - **Key**: `GITLAB_TOKEN`
   - **Value**: `glpat-xxxxxxxxxxxxxxxxxxxx`
   - **Flags**: ✅ Mask variable, ✅ Protect variable

## ⏱️ Estimativas de Tempo

| Projetos | Tempo estimado (1 token) |
|----------|--------------------------|
| 50       | ~3 min                   |
| 100      | ~6 min                   |
| 500      | ~25 min                  |
| 1.000    | ~55 min                  |
| 1.365    | ~1h10                    |

> O rate limit do GitLab SaaS é 300 req/min por usuário. O scanner respeita esse limite automaticamente via leitura dos headers `RateLimit-Remaining`.

## ⚠️ Troubleshooting

### Erro 401 Unauthorized
- **Causa**: Token inválido ou expirado.
- **Solução**: Gere um novo token em GitLab → User Settings → Access Tokens.

### Erro 403 Forbidden
- **Causa**: Token sem permissão de leitura no grupo/projeto.
- **Solução**: Verifique se o token tem escopo `read_api` e se o usuário tem acesso ao grupo.

### Erro 429 Too Many Requests
- **Causa**: Rate limit excedido.
- **Solução**: O scanner já trata isso automaticamente com retry + backoff. Se persistir, aguarde 60 segundos e execute novamente.

### Scanner interrompido no meio
- **Causa**: Queda de rede, máquina desligou, etc.
- **Solução**: Execute novamente. O checkpoint (`checkpoint.json`) garante que o scanner retoma de onde parou sem reprocessar projetos já escaneados.

### "Nao identificado" no Sistema Operacional
- **Causa**: Imagem Docker customizada ou não mapeada (ex: registry privado, imagens de ferramentas).
- **Solução**: Adicione a imagem ao dicionário `OS_MAP` no script.

## 📝 Notas Importantes

- **Checkpoint**: O scanner salva progresso automaticamente. Pode ser interrompido e retomado a qualquer momento sem perda de dados.
- **Excel Parcial**: A cada 50 projetos, um Excel intermediário é gerado (`dockerfiles_dpsp_parcial.xlsx`) como backup.
- **Idempotência**: O script pode ser executado múltiplas vezes com segurança. Projetos já processados são ignorados via checkpoint.
- **Rate Limit**: O rate limiter adaptativo lê os headers da API em tempo real e ajusta a velocidade automaticamente. Zero risco de bloqueio.
- **Todas as Branches**: O scanner varre todas as branches de cada projeto para cobertura completa (Prod, QA e Dev).

## 📄 Licença

Este script é fornecido "como está", sem garantias. Use por sua conta e risco.
