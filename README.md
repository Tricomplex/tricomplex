# Tricomplex

Tricomplex e uma plataforma para ajudar contadores e empresas a validarem a tributacao aplicada em notas fiscais eletronicas. O contador envia o XML de uma NF-e, o sistema extrai produto por produto, cruza NCMs e tributos declarados com uma base tributaria auditavel, e devolve uma analise clara com divergencias, valores esperados e fonte legal.

O objetivo nao e substituir o contador. O objetivo e reduzir o trabalho manual de procurar legislacao, conferir aliquota, interpretar mudancas e justificar decisoes com fonte oficial.

## Problema

Contadores precisam lidar com regras tributarias que mudam com frequencia. Para produtos, a dificuldade aumenta porque:

- cada produto pode ter NCM, descricao fiscal, CST/CSOSN, CFOP e regras especificas;
- cada estado pode publicar regras proprias;
- as fontes ficam espalhadas entre SEFAZ, Diario Oficial, leis, decretos, portarias, resolucoes e tabelas oficiais;
- produtos parecidos podem ter tratamentos diferentes;
- pequenas e medias empresas normalmente nao tem time dedicado para acompanhar tudo isso.

A proposta do Tricomplex e funcionar como um compilador de legislacao para regras tributarias: fontes oficiais entram de um lado, regras estruturadas e rastreaveis saem do outro.

## Escopo atual do MVP

Para reduzir complexidade, o MVP esta limitado a:

- UF principal: SP.
- NCMs:
  - `22011000`: agua mineral ou agua gaseificada
  - `22021000`: refrigerante / bebidas aromatizadas ou adicionadas de acucar
  - `20099000`: suco / misturas de sucos
  - `22030000`: cerveja
  - `84433233`: impressora
  - `84439923`: cartucho de impressora
  - `84439933`: toner de impressora
  - `48025610`: papel
  - `19059090`: alimento / produtos alimenticios de padaria/confeitaria
- Tributos no motor de analise:
  - ICMS
  - PIS
  - COFINS
  - IPI
- DIFAL e ICMS-ST podem aparecer no dominio, mas no MVP ficam como revisao/sem regra quando nao houver regra segura.

## Estrutura do repositorio

Este repositorio principal usa submodulos Git:

```text
tricomplex/
  back/
    extractor/   # extracao XML, comparacao fiscal, API FastAPI
    scraper/     # coleta/interpreta fontes oficiais e popula o MySQL
  front/         # interface web React/Vite
```

Submodulos:

```text
back/extractor -> https://github.com/Tricomplex/back.extractor.git
back/scraper   -> https://github.com/Tricomplex/back.scraper.git
front          -> https://github.com/Tricomplex/tricomplex-frontend.git
```

Mudancas dentro de `front`, `back/scraper` e `back/extractor` devem ser commitadas dentro do respectivo submodulo. Depois disso, o repositorio principal commita apenas o novo ponteiro do submodulo.

## Arquitetura

```text
                         fontes oficiais
                               |
                               v
                    back/scraper (local/job)
                               |
            valida JSON fiscal e grava regras
                               |
                               v
                         MySQL tributario
                               ^
                               |
usuario -> front React -> back/extractor FastAPI -> matcher deterministico
                                      |
                                      v
                              Gemini (texto amigavel)
                                      |
                                      v
                  JSON estruturado para o front renderizar
```

### Camada confiavel

A parte confiavel do sistema e deterministica:

- XML da NF-e e lido pelo `extractor.py`.
- Produto e casado por NCM, com fuzzy apenas como fallback/desempate.
- Regras sao buscadas no MySQL por NCM, tributo, jurisdicao e vigencia.
- Declarado vs esperado e comparado no `matcher.py`.
- Nenhuma LLM decide aliquota, status fiscal ou SQL.

### Camada de IA

A IA e usada em duas frentes diferentes:

- No `back/scraper`, pode ajudar a interpretar texto legal em JSON estruturado, que ainda e validado pelo codigo antes de ir ao banco.
- No `back/extractor`, Gemini transforma o resultado fiscal ja calculado em textos amigaveis, retornando JSON estruturado para o front.

Em ambos os casos, a IA nao deve inventar regra e nao deve gerar SQL executavel.

## Banco de dados

O schema atual foi pensado para regras versionadas e auditaveis.

Tabelas principais:

- `tributos`: ICMS, PIS, COFINS, IPI, DIFAL, ICMS-ST etc.
- `jurisdicoes`: federal, estadual ou municipal.
- `produtos_fiscais`: NCM, descricao, categoria e ativo.
- `fontes_legais`: fonte oficial, URL, orgao, data e trecho relevante.
- `regras_tributarias`: regra aplicavel, tipo, aliquota/pauta, vigencia, fonte e observacoes.

Principios:

- regra antiga nao deve ser sobrescrita;
- mudanca legal deve criar nova linha;
- `vigencia_inicio` e `vigencia_fim` representam o periodo real de validade;
- `ativo = 1` indica regra vigente/consideravel;
- cada regra deve ter fonte legal rastreavel.

## Fluxos principais

### Atualizar a base tributaria

```text
fontes oficiais
  -> scraping/download/parsing
  -> limpeza de contexto
  -> candidatos estruturados
  -> validacao deterministica
  -> inserts/upserts no MySQL
```

Normalmente esse fluxo pode rodar localmente de vez em quando, apontando para o banco em nuvem. Para o pitch, nao e obrigatorio deixar o scraper deployado.

### Analisar uma NF-e

```text
upload XML
  -> extracao de cabecalho, itens e impostos
  -> match por NCM
  -> busca de regras vigentes no MySQL
  -> comparacao declarado vs esperado
  -> Gemini escreve texto amigavel em JSON
  -> front renderiza resultado e permite baixar Markdown
```

## Rodar localmente

### 1. Banco

Tenha um MySQL acessivel com as tabelas do projeto e regras carregadas pelo scraper.

Variaveis comuns:

```text
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=
DB_NAME=tricomplex
```

### 2. Extractor/API

```bash
cd back/extractor
pip install -r requirements.txt
copy .env.example .env
uvicorn api:app --reload --host 0.0.0.0 --port 8000
```

No PowerShell, se estiver na raiz do repo principal, entre primeiro em `back/extractor`. Rodar `uvicorn api:app` na raiz nao funciona porque `api.py` fica dentro do submodulo extractor.

### 3. Front

```bash
cd front
npm install
copy .env.example .env
npm run dev
```

No `.env` do front:

```text
VITE_API_BASE_URL=http://localhost:8000
```

## Deploy sugerido para pitch

Para uma demo em que investidores abrem o site no celular:

- Front: Vercel ou Netlify.
- API FastAPI: Railway, Render ou Cloud Run.
- MySQL: banco gerenciado na nuvem, preferencialmente no mesmo provedor da API para reduzir latencia.
- Scraper: pode continuar rodando localmente quando necessario, apontando para o MySQL em nuvem.

Docker ajuda a empacotar a API, mas sozinho nao resolve hospedagem publica, HTTPS, banco persistente e variaveis de ambiente.

## Estado atual

Ja existe:

- Front React/Vite com upload de XML e exibicao da analise.
- API FastAPI em `back/extractor`.
- Extracao detalhada de XML de NF-e.
- Comparacao deterministica contra regras do MySQL.
- Resposta amigavel via Gemini em JSON estruturado.
- Scraper para fontes oficiais e integracao com MySQL.

Pontos ainda a evoluir:

- Golden dataset automatizado.
- Testes de integracao com banco real.
- Deploy publico.
- Job agendado para scraper, quando fizer sentido.
- Fluxo de revisao humana para regras ambiguas.

## Filosofia

A IA ajuda, mas nao e dona da verdade fiscal.

O nucleo confiavel precisa ser:

- auditavel;
- reexecutavel;
- validado por codigo;
- rastreavel ate fonte oficial;
- conservador quando houver ambiguidade.

Em resumo: a IA organiza texto; o backend valida; o banco guarda; a fonte prova.
