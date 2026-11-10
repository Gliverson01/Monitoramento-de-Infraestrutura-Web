# Monitoramento do site com SLO

[![ci](https://github.com/Gliverson01/Monitoramento-de-Infraestrutura-Web/actions/workflows/ci.yml/badge.svg)](https://github.com/Gliverson01/Monitoramento-de-Infraestrutura-Web/actions/workflows/ci.yml)
[![checagem-sintetica](https://github.com/Gliverson01/Monitoramento-de-Infraestrutura-Web/actions/workflows/synthetic-check.yml/badge.svg)](https://github.com/Gliverson01/Monitoramento-de-Infraestrutura-Web/actions/workflows/synthetic-check.yml)

Monitoramento sintético do meu site pessoal ([gliversonalmeida.com.br](https://www.gliversonalmeida.com.br/)) com
Prometheus, Blackbox Exporter e Grafana, um SLO definido e alertas por queima de orçamento de erro, com as regras
testadas em CI.

## O que tem aqui

| Peça | Para quê |
|---|---|
| `blackbox/` | Blackbox Exporter: acessa o site por HTTPS e mede sucesso, tempo por fase (DNS, TCP, TLS, servidor, transferência), código HTTP e validade do certificado. |
| `prometheus/prometheus.yml` | Sonda a home e o PDF do currículo a cada 30 s. |
| `prometheus/rules/slo.yml` | SLIs (recording rules), orçamento de erro e alertas. |
| `prometheus/tests/` | Testes das regras com `promtool test rules`: queda real dispara, falha isolada não. |
| `grafana/` | Datasource e dashboard provisionados: disponibilidade de 30 dias, orçamento restante, latência por fase, TLS. |
| `.github/workflows/ci.yml` | Valida a configuração, roda os testes das regras e sobe a stack inteira para conferir que a sondagem chega ao Prometheus e o dashboard ao Grafana. |
| `.github/workflows/synthetic-check.yml` | Acessa o site a cada 5 minutos pelo GitHub Actions. |

## SLO

| SLI | Objetivo | Janela |
|---|---|---|
| Disponibilidade: sondagens com HTTP 2xx em até 20 s | 99,5% (orçamento de ~3h36 por mês) | 30 dias |
| Latência: sondagens respondidas em menos de 1,5 s | 95% | 1 hora |

Alertas, no modelo de queima em múltiplas janelas do livro de SRE do Google:

| Alerta | Condição | Severidade |
|---|---|---|
| `SiteErrorBudgetFastBurn` | erro acima de 14,4x o permitido em 1h **e** nos últimos 5 min (gasta 2% do mês em 1h) | page |
| `SiteErrorBudgetSlowBurn` | erro acima de 6x o permitido em 6h **e** nos últimos 30 min (gasta 5% do mês em 6h) | ticket |
| `SiteDown` | sondagem falhando há 5 min | page |
| `SiteSlow` | menos de 95% das respostas abaixo de 1,5 s na última hora | ticket |
| `TLSCertExpiringSoon` | certificado vence em menos de 14 dias | ticket |
| `BlackboxExporterDown` | as sondagens pararam | ticket |

A janela curta existe para o alerta parar assim que o problema acaba, e a exigência de duas janelas evita acordar
alguém por uma falha isolada. O teste `slo_test.yml` cobre os dois casos.

## O que o monitor ensinou

O site roda no plano gratuito do Render, que desliga o serviço depois de ~15 minutos sem acesso. A primeira visita
depois disso leva de 30 a 60 s, e o Blackbox (timeout de 20 s) conta como falha, porque é o que o visitante sente.
No dashboard isso aparece como pico na fase `processing`. Sem acesso frequente, esse despertar sozinho consumiria o
orçamento de erro do mês. A checagem a cada 5 minutos do GitHub Actions mantém o serviço acordado e serve de alerta
quando o site cai (o GitHub avisa por e-mail quando o job falha).

## Como rodar

Requer Docker com Compose.

```bash
docker compose up -d
```

- Grafana: http://localhost:3000 (dashboard "Site: disponibilidade e SLO" na página inicial, acesso anônimo de leitura)
- Prometheus: http://localhost:9090 (alertas em `/alerts`)

Testes das regras sem instalar nada:

```bash
docker run --rm --entrypoint promtool -w /work/prometheus/tests -v "$PWD:/work:ro" prom/prometheus:v3.1.0 test rules slo_test.yml
```

## Próximos passos

- Alertmanager com envio para Telegram ou e-mail.
- Prometheus rodando continuamente numa VM pequena, para o SLO de 30 dias ter histórico real.
