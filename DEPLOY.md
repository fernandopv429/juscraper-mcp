# Deploy no Coolify

Este é um fork de `lab-dados/juscraper-mcp`. O upstream faz deploy em Azure
Container Apps (`.github/workflows/deploy.yml`); aqui o deploy é Coolify, pelo
`Dockerfile` da raiz. O workflow do Azure continua no repo mas está inerte —
Actions vem desligado em fork, e sem os secrets da Azure ele só falharia.

A instância pública do LabDados (a do README) estava fora do ar em 21/09/2026:
`404 — Container App is stopped or does not exist`. Daí este deploy próprio.

## 1. Criar o recurso

**New Resource → Application → Public Repository**

| Campo | Valor |
|---|---|
| Repositório | `https://github.com/fernandopv429/juscraper-mcp` |
| Branch | `main` |
| Build Pack | `Dockerfile` |
| Port | `8080` |
| Health Check Path | `/health` |

O build resolve as dependências com `uv` e instala o `juscraper` direto do
GitHub — leva alguns minutos e precisa de rede saindo do servidor.

## 2. Variáveis de ambiente

**Nenhuma é obrigatória.** Todo limite tem padrão em `juscraper_mcp/settings.py`.
Vale mexer em duas, porque os padrões foram calibrados para um serviço público
com orçamento de R$ 200/mês em nuvem — num servidor próprio eles são apertados
demais:

| Variável | Padrão | Observação |
|---|---|---|
| `JUSMCP_RATE_LIMIT_MAX` | `30` | requisições por IP a cada 60s. Atrás do proxy do Coolify o IP pode chegar igual para todo mundo; se o agente tomar 429, suba. |
| `JUSMCP_MAX_CONCORRENCIA` | `4` | scrapes simultâneos no servidor inteiro |
| `JUSMCP_MAX_LINHAS` | `300` | linhas por resposta |
| `JUSMCP_TIMEOUT_TOOL` | `200` | segundos até abortar uma tool call |
| `JUSMCP_SLEEP_TIME` | `1.0` | pausa entre requisições ao tribunal. **Não baixar** — é o que evita bloqueio no eSAJ. |

## 3. Conferir depois de subir

```bash
curl -s https://SEU-DOMINIO/health
```

Deve responder `{"status":"ok","service":"juscraper-mcp","version":"0.1.0"}`.

O endpoint MCP é `https://SEU-DOMINIO/mcp` (streamable HTTP, stateless):

```bash
claude mcp add --transport http juscraper https://SEU-DOMINIO/mcp
```

No n8n, o nó MCP Client aponta para a mesma URL. **Sem autenticação** — quem
souber a URL consulta. Se for ficar em domínio público, vale pôr basic auth no
proxy do Coolify.

## 4. O que funciona (verificado em 21/09/2026)

Seis das sete tools foram testadas contra os tribunais de verdade:
`listar_tribunais`, `buscar_jurisprudencia` (TJSP e TJRS), `buscar_julgados_
primeira_instancia` (CJPG), `datajud_contar_processos`, `datajud_listar_
processos` e `buscar_comunicacoes_cnj`.

`consultar_processo` precisou de conserto (o `cpopg` do juscraper mudou de
DataFrame para dict de seções). Hoje o 1º grau devolve dados básicos, partes e
petições. Duas limitações que vêm do pacote `juscraper`, não daqui:

- **`movimentacoes` volta sempre vazia** no 1º grau. A resposta avisa e sugere
  `peticoes_diversas` como aproximação do histórico.
- **2º grau (`cposg`) não extrai nada** — o download acontece, o parse volta
  vazio, inclusive em números válidos. A resposta avisa em vez de fingir que o
  processo não existe.

O CJPG devolveu vazio uma vez no meio de uma rajada de chamadas e voltou ao
normal em seguida: o TJSP estrangula. Vazio sem erro é resultado possível.

## 4.1 Varredura dos 24 tribunais (21/09/2026)

`buscar_jurisprudencia` com a mesma busca em todos os tribunais do catálogo,
a partir de um IP residencial brasileiro. **18 responderam, 6 não.**

Responderam com ementas: `tjsp` `tjac` `tjal` `tjam` `tjba` `tjce` `tjdft`
`tjes` `tjms` `tjpa` `tjpe` `tjpi` `tjpr` `tjrn` `tjrr` `tjsc` `tjrj` `tjrs`.

| Tribunal | O que acontece | Causa |
|---|---|---|
| `tjmt` | `JSONDecodeError` | O site está em manutenção: `jurisprudencia.tjmt.jus.br/assets/config/config.json` devolve HTML ("Site em manutenção — TJMT") no lugar do JSON de configuração. Passageiro. |
| `tjro` | `JSONDecodeError` | `juris-back.tjro.jus.br/search/varios` devolve a página "STIC — Página Bloqueada". Bloqueio do tribunal. |
| `tjap` | `403` | `tucujuris.tjap.jus.br` recusa mesmo com user-agent de navegador. Bloqueio no WAF. |
| `tjpb` | `403` | `pje-jurisprudencia.tjpb.jus.br`, idem. |
| `tjto` | `403` | `jurisprudencia.tjto.jus.br/consulta.php` recusa o user-agent do juscraper, mas responde `202` com user-agent de navegador — o bloqueio é por user-agent. |
| `tjgo` | 0 resultados, sem erro | O scraper do Projudi devolve DataFrame vazio, com e sem filtro de data. Falha silenciosa dentro do `juscraper`. |

Nenhuma dessas falhas vem do wrapper MCP — são o pacote `juscraper` e os
próprios tribunais. Repetidas 3 vezes, todas persistentes.

**Vale repetir a varredura do servidor do Coolify**: quatro dessas falhas são
bloqueio (`tjap`, `tjpb`, `tjro`, `tjto`) e podem depender do IP de origem. As
outras duas (`tjmt` em manutenção, `tjgo` vazio) falham de qualquer lugar.

O Datajud e o Comunica CNJ são APIs nacionais e cobrem todos os tribunais do
país — os dois responderam normalmente, inclusive para tribunais que o
scraping não alcança.

## 4.2 Varredura em produção (21/09/2026)

No ar em `https://juscraper.nexusdevhub.com` (Coolify, mesmo servidor do n8n).
Repetindo a varredura contra o deploy — ou seja, com os tribunais vendo o IP do
servidor em vez de um IP residencial: **16 dos 24 respondem**, contra 18 de
casa. As diferenças:

| Tribunal | De casa | Do servidor | Leitura |
|---|---|---|---|
| `tjes` | 20 ementas | `403` | **O IP do servidor está bloqueado.** Já falhou na primeira chamada, então é bloqueio prévio de faixa de datacenter, não consequência da varredura. |
| `tjpa` | 25 ementas (1ª varredura) | timeout | Passou a dar timeout **dos dois lugares**. É o tribunal, não o IP. |
| `tjap` | `403` genérico | mensagem clara | Do servidor o juscraper identifica a causa: **CAPTCHA Cloudflare Turnstile**. Mesmo caso do TJMG — fora do escopo por desenho, não é falha. |

Respondem em produção (16): `tjsp` `tjac` `tjal` `tjam` `tjba` `tjce` `tjdft`
`tjms` `tjpe` `tjpi` `tjpr` `tjrn` `tjrr` `tjsc` `tjrj` `tjrs`.

Não respondem (8): `tjap` (Turnstile), `tjes` (IP bloqueado), `tjmt`
(manutenção), `tjpa` (fora do ar), `tjpb` (403), `tjro` (bloqueio), `tjto`
(403), `tjgo` (vazio silencioso).

**Não baixe o `JUSMCP_SLEEP_TIME` nem suba a concorrência.** O `tjes` mostra o
que acontece com IP de servidor: uma vez na lista de bloqueio do tribunal, some
um tribunal inteiro do catálogo e não há como reverter do seu lado. De casa ele
funciona; do servidor, não mais.

`consultar_processo` foi conferido em produção no processo
`1108672-79.2023.8.26.0002`: devolve `formato: secoes` com dados básicos, 3
partes e 26 petições. É o build corrigido.

## 5. Atualizar

O `pyproject.toml` pina o `juscraper` num commit. O upstream apontava para
`@main`, e foi assim que o `consultar_processo` quebrou sozinho entre um build e
outro. Para subir a versão: troque o SHA, rode `pytest` e faça um teste real
antes de publicar.
