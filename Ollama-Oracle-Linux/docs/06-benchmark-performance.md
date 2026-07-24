# 6. Benchmark de performance (GPU — Tesla V100S)

Métricas a coletar por modelo: tokens/s, uso de VRAM, `GPU-Util`, latência de primeira
resposta e se o Flash Attention ficou `Enabled` ou `Disabled` para aquele modelo.

<!-- Os resultados reais medidos nesta V100S serão adicionados abaixo, nesta ordem cronológica -->

## 6.1 Referência: meça a velocidade real (tokens/s)

**Objetivo:** ter uma referência rápida de como medir e interpretar tokens/s a partir de qualquer chamada à API do Ollama, sem precisar rodar um teste dedicado toda vez.

Toda resposta da API `/api/generate` ou `/api/chat` traz métricas de performance. A conta que importa:

```
tokens/s = eval_count / (eval_duration em nanossegundos ÷ 1.000.000.000)
```

**Comandos:**
```bash
curl -s http://localhost:11434/api/generate -d '{
  "model": "<modelo>",
  "prompt": "Explique fotossíntese em um parágrafo.",
  "stream": false
}' | grep -o '"eval_count":[0-9]*\|"eval_duration":[0-9]*'
```

- `eval_count`: quantidade de tokens gerados na resposta (não inclui o processamento do prompt de entrada, contado separadamente em `prompt_eval_count`).
- `eval_duration`: tempo gasto gerando esses tokens, em **nanossegundos**.

## 6.2 O que registrar por modelo do elenco

Para cada modelo definido em [04-modelos-download-execucao.md](04-modelos-download-execucao.md), registrar nesta tabela (com dados reais, à medida que forem medidos):

| Modelo | Quantização | num_ctx | VRAM usada (`nvidia-smi`) | 100% GPU? (`ollama ps`) | Flash Attention | Tokens/s |
|---|---|---|---|---|---|---|
| _(preencher)_ | | | | | | |

**Por quê medir exatamente isso:**
- **VRAM usada** (`nvidia-smi`, ver [05-testes-validacao.md](05-testes-validacao.md)) confirma se ainda há folga para aumentar contexto ou manter outro modelo residente (`OLLAMA_MAX_LOADED_MODELS`, ver [03-configuracao.md](03-configuracao.md)).
- **100% GPU?** (`ollama ps`) é o primeiro sinal de alerta: qualquer split para CPU já explica uma queda de 3–5x no throughput antes mesmo de olhar tokens/s.
- **Flash Attention** por modelo é específico da V100 (suporte parcial, ver [01-prerequisitos-e-ambiente.md](01-prerequisitos-e-ambiente.md)) — sem registrar isso, um mesmo modelo pode parecer ter regredido de performance numa atualização quando na verdade o Flash Attention deixou de ativar para ele.

## 6.3 Comparando VRAM disponível vs. contexto

Como visto em [01-prerequisitos-e-ambiente.md](01-prerequisitos-e-ambiente.md):

```
VRAM total = pesos do modelo (quantização define)  +  KV cache (contexto define)  +  overhead
```

Ao testar um novo valor de `num_ctx` num Modelfile, meça a VRAM usada antes e depois com `nvidia-smi` — isso mostra o custo real do KV cache para aquele modelo, em vez de estimar de cabeça.
