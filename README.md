# Agendador por gatilho (amostra)

Tarefa agendada que roda de verdade, sem simulação: um workflow do GitHub Actions
dispara por `cron`, um script em Python busca uma notícia positiva no feed da
BBC News Brasil e cada execução fica registrada num log. Ninguém interage com
ela: roda sozinha no horário marcado.
No trabalho uso Airflow; aqui o agendador é o GitHub Actions.

## Sobre esta amostra

Este repositório é uma amostra pequena para validar o fluxo inteiro do
agendador: 4 execuções, de hora em hora, e uma página para visualizar o
resultado. Depois da quarta execução o próprio workflow se desliga.

A versão final, com uma notícia por dia às 08h de Brasília, fica em outro
repositório (`noticias-diaria`).

## Como funciona

1. O Actions dispara o workflow `diario.yml`.
2. `scripts/mensagem_do_dia.py` lê o feed RSS e descarta notícias negativas e
   políticas. As demais ganham pontos por palavras positivas.
3. Escolhe a de maior pontuação que ainda não foi usada.
4. Grava `data/hoje.json` e `data/log.json` na branch `dados`.
5. `index.html` lê o log e mostra as notícias.

A `main` só tem o código. A branch `dados` só tem commits automáticos do
workflow (`execução 01/04`).

## Resultados e erros

| Status | Quando | Saída |
|---|---|---|
| `OK` | achou uma notícia | 0 |
| `SEM_NOTICIA` | nada passou no filtro (o log traz a contagem) | 0 |
| `ERRO` | falha técnica (feed fora do ar, timeout...) | 1 |

## Airflow x GitHub Actions

| Airflow | GitHub Actions |
|---|---|
| DAG | workflow |
| schedule | `cron` |
| task | step |
| trigger manual | `workflow_dispatch` |
| retries | tentativas na busca do feed |

O Actions não é um orquestrador completo como o Airflow: não há dependências
entre tarefas, reprocessamento por data nem painel próprio de execuções.

## Rodar localmente

```bash
pip install -r requirements.txt pytest
pytest -q
python scripts/mensagem_do_dia.py
```

## Fonte

Notícias da [BBC News Brasil](https://www.bbc.com/portuguese). O projeto grava
apenas título, link e fonte, sempre com o link para a matéria original.
