# Ollama Mastering — Rodando LLMs localmente no Ubuntu (sem GPU)

Documentação passo a passo da configuração de um ambiente local para rodar modelos de LLM
com [Ollama](https://ollama.com) em **Ubuntu 22.04.5 LTS, apenas CPU (sem GPU)**.

O objetivo é registrar cada etapa real do processo — comandos, saídas, decisões e problemas
encontrados — para servir como referência reprodutível para a comunidade.

## Ambiente de referência

- SO: Ubuntu 22.04.5 LTS
- GPU: nenhuma (execução 100% CPU)
- Ollama: (preencher versão quando instalado)

## Estrutura da documentação

| Etapa | Arquivo | Conteúdo |
|---|---|---|
| 1 | [docs/01-prerequisitos-e-ambiente.md](docs/01-prerequisitos-e-ambiente.md) | Hardware, SO, dependências, checagens iniciais |
| 2 | [docs/02-instalacao-ollama.md](docs/02-instalacao-ollama.md) | Instalação do Ollama, serviço systemd |
| 3 | [docs/03-configuracao.md](docs/03-configuracao.md) | Variáveis de ambiente, diretórios, configs |
| 4 | [docs/04-modelos-download-execucao.md](docs/04-modelos-download-execucao.md) | Pull/run de modelos, Modelfile |
| 5 | [docs/05-testes-validacao.md](docs/05-testes-validacao.md) | Smoke tests, chamadas via API/CLI |
| 6 | [docs/06-benchmark-performance.md](docs/06-benchmark-performance.md) | Tokens/s, RAM, CPU, latência por modelo |
| 7 | [docs/07-troubleshooting.md](docs/07-troubleshooting.md) | Erros encontrados e soluções |

## Como este repositório é alimentado

Cada etapa é registrada no formato:

````
### <título da etapa>

**Objetivo:** o que essa etapa resolve/configura

**Comandos:**
```bash
comando aqui
```

**Saída / observações:**
resultado relevante, decisões tomadas, avisos

**Problemas encontrados:** (se houver)
````
