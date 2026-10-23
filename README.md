# Agendador

Workflow agendado do GitHub Actions que integra um feed RSS externo (BBC News
Brasil) a uma rotina em Python e registra cada execução num log. Este
repositório é a amostra que validou o fluxo de ponta a ponta: 4 execuções
agendadas, após as quais o próprio workflow desliga o agendamento. A versão
final, com uma execução por dia, está em
[noticias-diaria](https://github.com/CassiaCristina0o/noticias-diaria).

## Integração com o feed

`scripts/mensagem_do_dia.py` consome o feed RSS da BBC News Brasil.

- **Requisição:** `requests` com timeout de 10 s e `User-Agent` identificado.
- **Tentativas:** até 3, com 30 s de espera, apenas para falhas passageiras
  (timeout, erro de conexão e respostas 5xx). Respostas 4xx, como 404, não são
  repetidas.
- **Leitura:** `feedparser` extrai título, link, resumo e miniatura de cada item.
  O resumo serve apenas à seleção e não é gravado.
- **Erros tipificados:** cada falha vira um `ErroDoProcesso` com etapa, tipo e
  mensagem (`Timeout`, `ConnectionError`, `HTTPError 503`, `FeedVazio`), que vai
  para o log.
- **Configuração:** a URL do feed vem da variável `FEED_URL`, o que permite
  provocar uma falha controlada para testar o alerta.

Os dados gravados são título, link, fonte e o endereço da miniatura oferecido
pelo próprio feed. A imagem não é copiada.

## Execução agendada

`.github/workflows/diario.yml`:

1. `cron` de hora em hora (`7 * * * *`), mais `workflow_dispatch` com o campo
   `feed_url`.
2. `concurrency` impede duas execuções simultâneas.
3. O script grava `data/hoje.json` e `data/log.json` numa branch própria, a
   `dados`, com um commit por execução (`execução 01/04`). A `main` contém só o
   código.
4. O GitHub envia um e-mail a cada execução do workflow, com o status. Se o
   script termina com erro, o workflow registra o log e é marcado como falho, e
   o e-mail avisa a falha.
5. Na quarta execução, o workflow desliga o próprio agendamento.

`index.html` lê o log da branch `dados` e exibe as notícias e as execuções.

## Resultados e erros

| Status | Quando | Saída |
|---|---|---|
| `OK` | uma notícia foi escolhida | 0 |
| `SEM_NOTICIA` | nada passou no filtro (o log traz a contagem por motivo) | 0 |
| `ERRO` | falha técnica (feed indisponível, timeout, log ilegível) | 1 |

## Resultado da amostra

As 4 execuções ocorreram por agendamento e terminaram com sucesso. O `cron` foi
configurado para rodar a cada hora, mas o GitHub não garante o horário de
workflows agendados: os intervalos observados foram de cerca de 6 horas.

## Airflow x GitHub Actions

| Airflow | GitHub Actions |
|---|---|
| DAG | workflow |
| schedule | `cron` |
| task | step |
| trigger manual | `workflow_dispatch` |
| retries | tentativas na busca do feed |
| alerta de falha | e-mail do GitHub em execução com erro |

O Actions não é um orquestrador completo como o Airflow: não há dependências
entre tarefas, reprocessamento por data nem painel próprio de execuções.

## Executar localmente

```bash
pip install -r requirements.txt pytest
pytest -q
python scripts/mensagem_do_dia.py
```

## Fonte

Notícias da [BBC News Brasil](https://www.bbc.com/portuguese), sempre com o link
para a matéria original.
