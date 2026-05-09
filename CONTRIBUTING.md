# 🤝 Guia de Contribuição

Bem-vindo! Este documento descreve como contribuir para o projeto Highlights App, especialmente no que diz respeito ao versionamento de dados com DVC e fluxo de trabalho com Git.

---

## 📋 Configuração Inicial

### 1. Clone o repositório e recupere os dados

```bash
git clone https://github.com/volneiklehm/highlights-app.git
cd highlights-app
git checkout dvc  # Trabalhe no branch dvc
dvc pull          # Restaura todos os arquivos .parquet
pip install -r requirements.txt
```

### 2. Verifique que os dados estão disponíveis

```bash
ls data/raw/               # Deve ter: news_category_raw.parquet
ls data/processed/         # Deve ter: X_train.parquet, X_test.parquet, y_train.parquet, y_test.parquet
dvc status                 # Deve exibir: "Data and pipelines are up to date"
```

---

## 🔄 Fluxo de Trabalho com Dados (DVC)

### Cenário 1: Você alterou a lógica de preprocessamento

Se você modificou `src/preprocessing.py` e quer gerar novos splits:

```bash
# 1. Altere o arquivo conforme necessário
vim src/preprocessing.py

# 2. Re-processe os dados
python src/preprocessing.py

# 3. Rastreie as alterações com DVC
dvc add data/processed/X_train.parquet data/processed/X_test.parquet \
       data/processed/y_train.parquet data/processed/y_test.parquet

# 4. Verifique as mudanças
git status  # Verá: X_train.parquet.dvc (modificado), .gitignore

# 5. Commit do Git (apenas os metadados)
git add data/processed/*.dvc data/processed/.gitignore
git commit -m "refactor: update preprocessing logic and regenerate splits"

# 6. Push para DVC remote (DagsHub)
dvc push

# 7. Push para Git
git push origin dvc
```

### Cenário 2: Você alterou a ingestão de dados (Supabase)

Se você modificou `src/ingestion.py` e quer re-ingerir dados do Supabase:

```bash
# 1. Altere src/ingestion.py conforme necessário
vim src/ingestion.py

# 2. Re-ingira os dados
python src/ingestion.py

# 3. Rastreie a camada raw
dvc add data/raw/news_category_raw.parquet

# 4. Reprocesse os dados
python src/preprocessing.py

# 5. Rastreie os splits atualizados
dvc add data/processed/X_train.parquet data/processed/X_test.parquet \
       data/processed/y_train.parquet data/processed/y_test.parquet

# 6. Commit e push
git add data/raw/*.dvc data/raw/.gitignore data/processed/*.dvc data/processed/.gitignore
git commit -m "chore: re-ingest data from Supabase and regenerate splits"
dvc push
git push origin dvc
```

### Cenário 3: Você clonó repo e precisa trabalhar com dados existentes

```bash
# Após clonar e estar no branch dvc
dvc pull

# Agora você tem todos os .parquet localmente e pode:
python src/train.py       # Treinar com os dados existentes
python src/evaluate.py    # Avaliar o melhor modelo
```

---

## 🌳 Fluxo de Branches

### Branch `dvc` (principal para dados)
- Contém os arquivos `.dvc` pointer files
- Contém `.dvc/config` (remote configuration)
- Contém `.dvc/config.local` (credenciais - **nunca commitar**)
- É o branch padrão para trabalhar com dados

### Branch `main` (opcional, para releases)
- Pode conter versão estável do código
- Dados podem ser referenciados do branch `dvc`

**Fluxo recomendado:**
```
dvc branch (dados versionados)
   ├── Merge/trabalho do preprocessamento
   ├── Merge/trabalho da ingestão
   └── Merge/atualizações de modelo
```

---

## ✅ Checklist Antes de Fazer Push

Antes de commitar e fazer push, verifique:

```bash
# 1. Status do DVC (garantir que tudo está atualizado)
dvc status

# 2. Status do Git (não deixar arquivos .parquet sem rastreamento)
git status
# ✓ Não deve haver .parquet files listados como "Untracked" ou "Modified"
# ✓ Apenas .dvc files e .gitignore devem aparecer

# 3. Configuração de remote
dvc remote list
# ✓ Deve exibir: origin  https://dagshub.com/volneiklehm/highlights-app.dvc

# 4. Verificar credenciais (config.local)
cat .dvc/config.local
# ✓ Deve conter: auth = basic, user = e-sardinha (seu usuário do DagsHub)

# 5. Sincronizar com remote
dvc push    # Push dos dados para DagsHub
dvc status --cloud
# ✓ Deve exibir: "Cache and remote are in sync"

# 6. Fazer o commit do Git
git add .dvc data/**/*.dvc data/**/.gitignore
git commit -m "chore: sync data updates with DVC"
git push origin dvc
```

---

## 🔐 Variáveis de Ambiente (.env)

O arquivo `.env` na raiz do projeto contém suas credenciais pessoais. **Nunca commitar esse arquivo.**

Certifique-se de que está no `.gitignore`:
```bash
echo ".env" >> .gitignore
git add .gitignore
git commit -m "chore: ensure .env is ignored"
```

Seu `.env` local deve ter:
```env
DAGSHUB_USER=seu-usuario-dagshub
DAGSHUB_REPO=highlights-app
DAGSHUB_TOKEN=seu-token-dagshub
SUPABASE_URL=sua-url-supabase
SUPABASE_KEY=sua-chave-supabase
```

---

## 🚨 Troubleshooting

### "ERROR: output 'data/processed/X_train.parquet' is already tracked by SCM (Git)"

Significado: O arquivo `.parquet` foi commitado no Git, mas DVC quer rastreá-lo.

**Solução:**
```bash
git rm --cached data/processed/X_train.parquet
dvc add data/processed/X_train.parquet
git add data/processed/X_train.parquet.dvc data/processed/.gitignore
git commit -m "chore: move X_train.parquet to DVC tracking"
```

### "dvc pull" não restaura os arquivos

Possíveis causas:
1. Credenciais (`config.local`) incorretas
2. Token do DagsHub expirou
3. Remote não está apontando para o repositório correto

**Verificar:**
```bash
dvc remote list  # Deve ser: https://dagshub.com/volneiklehm/highlights-app.dvc
dvc push -v      # Com verbose, vê detalhes da falha
```

### "dvc status --cloud" trava

```bash
# Interrompa (Ctrl+C) e tente com timeout
timeout 10 dvc status --cloud
```

---

## 📚 Recursos Adicionais

- **DVC Documentation**: https://dvc.org/doc
- **DagsHub**: https://dagshub.com
- **Conventional Commits**: https://www.conventionalcommits.org/

---

## 👥 Dúvidas?

Se tiver dúvidas ou encontrar problemas, abra uma issue no repositório GitHub ou converse com o time.

Obrigado por contribuir! 🎉
