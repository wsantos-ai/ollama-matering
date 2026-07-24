# Ollama Mastering — Ambiente otimizado em GPU Tesla V100S (Oracle Linux)

Documentação passo a passo da configuração de um ambiente agêntico local para rodar
modelos de LLM com [Ollama](https://ollama.com) em **Oracle Linux Server 9.8, com GPU NVIDIA
Tesla V100S (32 GB VRAM)**.

O objetivo é registrar cada etapa real do processo — comandos, saídas, decisões e problemas
encontrados — para servir como referência reprodutível para a comunidade, incluindo os
conceitos fundamentais (quantização, offload, KV cache, Flash Attention) necessários para
entender **por que** cada ajuste é feito e não só copiar comandos.

## Ambiente de referência

- SO: Oracle Linux Server 9.8 (`VERSION_ID="9.8"`, `PLATFORM_ID="platform:el9"`)
- GPU: NVIDIA Tesla V100S, arquitetura Volta (Compute Capability 7.0), 32 GB HBM2
- CUDA Toolkit: 12.8
- Ollama: (preencher versão quando instalado)

## Estrutura da documentação

| Etapa | Arquivo | Conteúdo |
|---|---|---|
| 1 | [docs/01-prerequisitos-e-ambiente.md](docs/01-prerequisitos-e-ambiente.md) | Fundamentos (quantização, offload, KV cache, Flash Attention), ficha técnica da V100S, preparação do SO |
| 2 | [docs/02-instalacao-ollama.md](docs/02-instalacao-ollama.md) | Driver NVIDIA, CUDA Toolkit, instalação do Ollama e serviço systemd |
| 3 | [docs/03-configuracao.md](docs/03-configuracao.md) | Variáveis de ambiente de otimização, override do systemd, firewall |
| 4 | [docs/04-modelos-download-execucao.md](docs/04-modelos-download-execucao.md) | Elenco de modelos por papel, pull/run, Modelfiles |
| 5 | [docs/05-testes-validacao.md](docs/05-testes-validacao.md) | Smoke tests, `ollama ps`, logs, validação do Flash Attention |
| 6 | [docs/06-benchmark-performance.md](docs/06-benchmark-performance.md) | Tokens/s, VRAM, GPU-Util, latência por modelo |
| 7 | [docs/07-troubleshooting.md](docs/07-troubleshooting.md) | Erros específicos da V100S/Oracle Linux e checklist final |

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
