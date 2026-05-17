# Tricomplex

Tricomplex e uma plataforma para ajudar contadores e empresas a validarem a tributacao aplicada em notas fiscais. A ideia central e simples: o contador envia o XML de uma NF-e, o sistema le produto por produto, compara os tributos declarados na nota com uma base tributaria atualizada por fontes oficiais, e devolve uma analise clara:

> "Nesta nota fiscal, este produto parece estar com estes tributos incorretos. O valor correto seria X. A regra veio desta fonte oficial. Os riscos de manter como esta sao Y."

O foco do produto nao e substituir o contador. O foco e reduzir o trabalho manual de procurar legislacao, conferir aliquota, interpretar mudancas e justificar decisoes com fonte oficial.

## Contexto do problema

Contadores precisam lidar com uma quantidade grande de regras tributarias que mudam com frequencia. Para produtos, a dificuldade aumenta porque:

- cada produto pode ter NCM, descricao fiscal, CST/CSOSN, CFOP e regras especificas;
- cada estado pode publicar regras proprias;
- algumas regras estao em SEFAZ, outras em Diario Oficial, leis, decretos, portarias, resolucoes ou tabelas;
- nem sempre a fonte usa uma estrutura facil de processar;
- produtos parecidos podem ter tratamentos diferentes;
- empresas pequenas e medias normalmente nao tem time dedicado para acompanhar tudo isso.

Na entrevista/descoberta do problema, a dor mais forte foi a busca e interpretacao de fontes legais atualizadas. A parte mais valiosa para o contador e saber:

- qual aliquota ou pauta deveria ser aplicada;
- de onde veio essa regra;
- por que ela vale para aquele produto;
- qual risco existe em deixar a nota como esta.

## Escopo atual do MVP

Para reduzir a complexidade inicial, o MVP esta limitado a:

- UF: SP
- NCMs:
  - `22011000`: agua mineral ou agua gaseificada
  - `22021000`: refrigerante / bebidas aromatizadas ou adicionadas de acucar
  - `20099000`: suco / misturas de sucos
- Tributos:
  - ICMS
  - DIFAL
  - PIS
  - COFINS
  - IPI
  - ICMS-ST

O objetivo agora e deixar o motor de scraping/atualizacao da base funcionando bem para esse recorte pequeno antes de expandir para mais NCMs, UFs e tributos.

## Estrutura do repositorio

Este repositorio principal usa submodulos Git:

```text
tricomplex/
  back/
    extractor/   # submodulo: extracao e analise de XML de NF-e
    scraper/     # submodulo: scraping, interpretacao legal e carga no banco
  front/         # submodulo: interface web
```

Submodulos configurados:

```text
back/extractor -> https://github.com/Tricomplex/back.extractor.git
back/scraper   -> https://github.com/Tricomplex/back.scraper.git
front          -> https://github.com/Tricomplex/tricomplex-frontend.git
```

Importante: mudancas dentro de `front`, `back/scraper` e `back/extractor` devem ser commitadas dentro do repositorio do respectivo submodulo. Depois disso, o repositorio principal deve commitar apenas o novo ponteiro do submodulo.

## Arquitetura conceitual

Existem dois motores principais.

### 1. Motor de atualizacao tributaria

Esse motor roda de tempos em tempos.

Fluxo esperado:

```text
fontes oficiais
  -> scraping / download / parsing
  -> limpeza e recorte de trechos relevantes
  -> interpretacao por LLM em JSON estruturado
  -> validacao deterministica pelo backend
  -> upsert/insercao no banco tributario
  -> historico auditavel de regras
```

Esse e o motor de prioridade absoluta no momento.

Ele deve buscar fontes como:

- SEFAZ SP;
- Diario Oficial;
- leis;
- decretos;
- portarias;
- resolucoes;
- tabelas oficiais, como TIPI em XLSX.

O resultado final nao deve ser SQL gerado pela LLM. A LLM deve retornar JSON estruturado. O codigo valida esse JSON e so depois grava no banco.

### 2. Motor de analise de NF-e

Esse motor roda quando o usuario envia uma nota.

Fluxo esperado:

```text
upload XML NF-e
  -> extracao de cabecalho, produtos e impostos declarados
  -> match de produto/NCM contra a base tributaria
  -> busca das regras vigentes aplicaveis
  -> comparacao entre imposto declarado e imposto esperado
  -> resposta estruturada
  -> LLM opcional para deixar a mensagem clara e bonita
  -> front exibe resultado para o contador
```

Para NF-e, o XML e a fonte mais confiavel. Imagem, PDF ou DANFE visual nao trazem todos os campos necessarios.

## Banco de dados

O schema atual foi pensado para armazenar regras tributarias versionadas e auditaveis.

Tabelas principais:

- `tributos`: ICMS, DIFAL, PIS, COFINS, IPI, ICMS-ST etc.
- `jurisdicoes`: federal, estadual ou municipal.
- `produtos_fiscais`: NCM, descricao fiscal e categoria.
- `fontes_legais`: fonte oficial, URL, orgao, data e trecho relevante.
- `regras_tributarias`: regra aplicavel, tipo, aliquota/pauta, vigencia e fonte.

Principios importantes:

- regra antiga nao deve ser sobrescrita;
- uma mudanca legal deve criar nova linha;
- `vigencia_inicio` e `vigencia_fim` representam o periodo real de validade;
- `ativo = 1` indica que a regra esta valendo agora;
- `substitui_regra_id` pode apontar para regra anterior;
- cada regra deve ter fonte legal rastreavel.

## Motor de scraping

O submodulo `back/scraper` contem o pipeline de atualizacao tributaria.

Arquivos principais:

```text
back/scraper/
  fontes_mvp.json       # escopo fiscal e fontes oficiais iniciais
  urls.txt              # lista simples de URLs principais
  scraping.py           # scraper/crawler HTML generico
  limpador_dados.py     # extrai contexto ao redor de aliquotas e termos fiscais
  xlsx_extractor.py     # extrai linhas de fontes XLSX oficiais, como TIPI
  llm_interpreter.py    # chama LLM e valida JSON fiscal retornado
  db_integrator.py      # grava no MySQL usando o schema da Tricomplex
  pipeline.py           # orquestrador do fluxo inteiro
  .env.example          # exemplo de configuracao local
```

Fontes iniciais do MVP:

- RICMS/SP, artigos 52 a 56-C: aliquotas de ICMS
- Portaria CAT 68/2019: substituicao tributaria por segmento
- TIPI XLSX da Receita Federal: IPI por NCM
- Lei 10.637/2002: PIS
- Lei 10.833/2003: COFINS

## Como rodar o scraper

Entre no submodulo:

```bash
cd back/scraper
```

Instale dependencias:

```bash
pip install -r requirements.txt
```

Crie o `.env` a partir do exemplo:

```bash
cp .env.example .env
```

No Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

### Rodar apenas scraping e limpeza

```bash
python pipeline.py --no-llm --max-depth 0
```

Esse modo nao usa LLM e nao toca no banco. Ele gera artefatos em:

```text
back/scraper/artifacts/<timestamp>/
```

Artefatos principais:

- `*_scraped.json`: conteudo bruto estruturado;
- `*_contexts.json`: trechos perto de aliquotas e termos fiscais;
- `summary.json`: resumo da rodada.

### Rodar com LLM, sem banco

```bash
python pipeline.py
```

Esse modo exige `OPENAI_API_KEY` no `.env` enquanto o provider atual for OpenAI.

### Rodar com LLM e testar banco sem gravar

```bash
python pipeline.py --apply-db
```

Por padrao, `--apply-db` roda em dry-run: abre transacao, tenta resolver IDs/inserts e faz rollback.

### Gravar de verdade no banco

```bash
python pipeline.py --apply-db --commit
```

Use esse modo apenas quando o JSON interpretado estiver validado.

## LLM

O codigo atual esta preparado para OpenAI Responses API por padrao:

```text
OPENAI_API_KEY=
OPENAI_MODEL=gpt-4.1-mini
OPENAI_BASE_URL=https://api.openai.com/v1
```

Nenhuma LLM real foi usada durante a implementacao inicial do pipeline. O teste executado ate agora foi scraping + limpeza + TIPI XLSX.

Para custo menor, faz sentido implementar um provider Gemini. A recomendacao e manter o contrato interno identico:

```text
fonte + texto extraido -> LLM -> JSON estruturado -> validacao -> banco
```

Modelos gratuitos/baratos, como Gemini Flash, podem ser bons para o MVP, desde que:

- a LLM nunca gere SQL para execucao direta;
- o JSON seja validado pelo codigo;
- regras ambiguas fiquem pendentes ou com baixa confianca;
- sempre exista fonte e trecho relevante.

## Contrato esperado da LLM

A LLM deve retornar JSON puro, sem markdown, no formato:

```json
{
  "source": {
    "tipo": "SEFAZ",
    "titulo": "Fonte oficial",
    "url": "https://...",
    "orgao": "SEFAZ SP",
    "data_publicacao": "2025-01-01",
    "texto_relevante": "Trecho que sustenta a regra"
  },
  "rules": [
    {
      "tributo": "ICMS",
      "jurisdicao": {
        "tipo": "ESTADUAL",
        "uf": "SP",
        "municipio": null,
        "codigo_ibge": null
      },
      "produto": {
        "ncm": "22021000",
        "descricao": "Refrigerantes",
        "categoria": "Bebidas"
      },
      "tipo_regra": "ALIQUOTA",
      "aliquota_percentual": 18.0,
      "valor_fixo": null,
      "unidade_valor": null,
      "vigencia_inicio": "2025-01-01",
      "vigencia_fim": null,
      "ativo": true,
      "resumo_regra": "Aliquota aplicavel conforme fonte oficial.",
      "observacoes": null,
      "confianca": "ALTA"
    }
  ]
}
```

Campos que o validador confere:

- NCM dentro do escopo do MVP;
- tributo permitido;
- UF SP para regra estadual;
- tipo de regra valido;
- aliquota obrigatoria para `ALIQUOTA`;
- valor fixo obrigatorio para `PAUTA`;
- vigencia obrigatoria;
- fonte com tipo, titulo e URL.

## Golden standard

Para testar se o scraper inteiro esta correto, o projeto precisa de um golden dataset.

Golden dataset e uma base pequena, feita manualmente e tratada como verdade. O pipeline deve gerar o mesmo resultado quando processar as fontes.

Sugestao de estrutura:

```text
back/scraper/tests/golden/
  mvp_sp_bebidas_expected.json
  mvp_sp_bebidas_expected.sql
  fixtures/
    tipi_trecho.xlsx
    ricms_art052.html
    portaria_cat_68.html
```

O golden nao deve comparar IDs auto incrementais. Ele deve comparar campos de negocio:

- tributo;
- jurisdicao;
- UF;
- NCM;
- descricao fiscal;
- tipo de regra;
- aliquota percentual;
- valor fixo;
- unidade;
- vigencia;
- URL da fonte;
- trecho relevante.

O primeiro golden ideal deve cobrir, para o recorte SP + 3 NCMs:

- IPI para `20099000`, `22011000` e `22021000` via TIPI XLSX;
- ICMS geral interno de SP via RICMS, aplicado aos 3 NCMs com observacao de que vale salvo excecoes especificas;
- PIS e COFINS gerais nao cumulativos, aplicados aos 3 NCMs com observacao de regime/excecoes;
- ICMS-ST e DIFAL como itens de revisao enquanto nao houver aliquota/MVA/vigencia especifica segura.

Regra pratica: se a LLM nao conseguir justificar com trecho oficial, a regra nao entra como verdade no golden.

## Estado atual do projeto

### Ja existe

- Front em React/Vite com tela de upload e resultado mockado.
- Extractor Python para ler XML de NF-e.
- Matcher inicial contra banco.
- Scraper HTML generico.
- Limpador de contextos fiscais.
- Pipeline MVP do scraper.
- Extraidor XLSX para TIPI.
- Integrador MySQL baseado no schema informado.
- README do scraper com comandos principais.

### Pontos de atencao conhecidos

- O `matcher.py` antigo espera coluna `ean` em `produtos_fiscais`, mas o schema atual nao tem essa coluna.
- O `extractor.py` ainda precisa garantir extracao de NCM por item.
- O front ainda parece depender de mock de API.
- O provider Gemini ainda nao foi implementado.
- Falta golden dataset automatizado.
- Falta decidir fluxo de revisao humana para regras de confianca media/baixa.

## Proximos passos recomendados

1. Criar golden dataset do MVP.
2. Implementar testes que comparam saida do pipeline com o golden.
3. Implementar provider Gemini mantendo o mesmo contrato JSON.
4. Rodar LLM em cima das fontes ja coletadas.
5. Validar dry-run no MySQL local.
6. Ajustar schema se for necessario incluir dados como EAN/GTIN.
7. Integrar front com API real de analise.
8. Transformar o scraper em job agendado.

## Filosofia do sistema

O sistema ideal e um compilador de legislacao para regras tributarias.

A IA ajuda a interpretar texto dificil, mas nao deve ser dona do banco. A parte confiavel do sistema precisa ser:

- auditavel;
- reexecutavel;
- validada por codigo;
- rastreavel ate a fonte oficial;
- conservadora quando houver ambiguidade.

Em outras palavras: a LLM sugere, o backend valida, o banco guarda a regra e a fonte prova.
