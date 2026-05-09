# Plano de Migracao para DVC

## Objetivo
Migrar versionamento de dados para DVC de forma incremental, segura e sem travar o fluxo atual do time.

## Principios
- Primeiro estabilizar o fluxo atual.
- Migrar em fases pequenas.
- Garantir reproducibilidade para qualquer membro do time.
- Evitar migracao big-bang.

## Fase 0 - Congelamento do Estado Atual
1. Fechar um commit com o pipeline atual funcionando (ingestion, preprocessing, train, evaluate).
2. Manter os dados atuais no Git temporariamente para nao quebrar onboarding.
3. Definir um ponto de corte: a partir deste commit, novos artefatos seguem DVC.

### Criterio de saida
- Todos os membros conseguem rodar o fluxo atual localmente.

## Fase 1 - DVC Basico para Parquets
### Escopo desta fase
- data/raw/news_category_raw.parquet
- data/processed/X_train.parquet
- data/processed/X_test.parquet
- data/processed/y_train.parquet
- data/processed/y_test.parquet

### Fora do escopo nesta fase
- data/News_Category_Dataset_v3.json
- data/News_Category_Dataset_balanced.csv

### Entregaveis
1. Repositorio DVC inicializado.
2. Remote DagsHub configurado.
3. Arquivos .dvc criados para os parquets.
4. Dados enviados com dvc push.
5. Git passa a versionar metadados .dvc, nao os arquivos .parquet.

### Criterio de saida
- Clone novo + git pull + dvc pull restaura os mesmos parquets.

## Fase 2 - Padronizacao de Operacao do Time
1. Definir procedimento oficial para atualizar dados:
   - gerar dados
   - dvc add
   - dvc push
   - git commit dos .dvc
   - git push
2. Definir procedimento oficial para consumir dados:
   - git pull
   - dvc pull
3. Revisar .gitignore para impedir commit acidental de .parquet.
4. Adicionar checklist de PR para mudancas de dados.

### Criterio de saida
- Pelo menos 2 pessoas do time executam o fluxo sem suporte manual.

## Fase 3 - Introducao do dvc.yaml
### Valor esperado
- Reprodutibilidade com estagios claros.
- Execucao incremental com dvc repro.
- Menos erro humano na ordem dos scripts.

### Estagios sugeridos
1. ingest: usa src/ingestion.py e gera data/raw/news_category_raw.parquet
2. preprocess: usa src/preprocessing.py e gera data/processed/*.parquet e data/processed/split_meta.json
3. train: usa src/train.py e gera artefatos de treino
4. evaluate: usa src/evaluate.py e valida o modelo

### Criterio de saida
- Execucao do pipeline com comando unico de reproducao.
- Mudanca em script ou dado dispara apenas estagios necessarios.

### Exemplo de dvc.yaml
```yaml
stages:
   ingest:
      cmd: python src/ingestion.py
      deps:
         - src/ingestion.py
         - .env
      outs:
         - data/raw/news_category_raw.parquet

   preprocess:
      cmd: python src/preprocessing.py
      deps:
         - src/preprocessing.py
         - data/raw/news_category_raw.parquet
      outs:
         - data/processed/X_train.parquet
         - data/processed/X_test.parquet
         - data/processed/y_train.parquet
         - data/processed/y_test.parquet
         - data/processed/split_meta.json

   train:
      cmd: python src/train.py
      deps:
         - src/train.py
         - src/preprocessing.py
         - data/processed/X_train.parquet
         - data/processed/y_train.parquet
         - data/processed/X_test.parquet
         - data/processed/y_test.parquet
      outs:
         - model.pkl

   evaluate:
      cmd: python src/evaluate.py
      deps:
         - src/evaluate.py
         - model.pkl
         - data/processed/X_test.parquet
         - data/processed/y_test.parquet
```

Observacoes:
- O uso de .env em deps e opcional e pode ser removido se nao quiser que mudancas de credenciais disparem o stage ingest.
- Caso prefira manter a avaliacao dentro do train.py, o stage evaluate pode ser removido.

## Fase 4 - Decisao sobre JSON/CSV Grandes
1. Se data/News_Category_Dataset_v3.json crescer ou mudar frequentemente, migrar para DVC.
2. Se for arquivo estatico de referencia, pode permanecer no Git por mais tempo.

### Criterio de saida
- Politica definida para arquivos grandes de origem.

## Riscos e Mitigacoes
1. Risco: esquecer dvc pull e executar pipeline sem dados locais.
   Mitigacao: instrucoes curtas no README e checagens de existencia de dados.
2. Risco: credenciais DagsHub em local incorreto.
   Mitigacao: usar configuracao local e nunca commitar segredo.
3. Risco: arquivos binarios voltarem ao Git apos migracao.
   Mitigacao: ajustar .gitignore e revisar PRs.

