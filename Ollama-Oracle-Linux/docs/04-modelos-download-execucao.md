# 4. Modelos: escolha, download e execução (Modelfiles)

Registro da estratégia de elenco de modelos por papel (roteador + especialistas) e dos
Modelfiles usados para fixar `num_ctx`, `num_gpu` e outros parâmetros por tarefa.

## 4.1 Escolhendo os modelos por tarefa

Num ambiente agêntico, você não quer um único modelo genérico — quer o modelo *certo* para cada etapa. A ideia é ter um **roteador leve** que planeja e decide, e **especialistas** que executam.

### Um ponto crítico: tool calling

Se seus agentes chamam ferramentas (functions/tools), a qualidade do *tool calling* varia muito entre famílias de modelo. Modelos das famílias Qwen, Llama e Mistral tendem a ter tool calling robusto; alguns modelos (como o Gemma) são notoriamente mais fracos nisso. **Escolha modelos com bom suporte a ferramentas para os papéis que interagem com tools.**

### Sugestão de elenco (dentro de 32 GB, faixas de tamanho)

| Papel no agente | Perfil de modelo | Quantização sugerida | Observação |
|---|---|---|---|
| Roteador / planejador | Pequeno, 7–8B, bom em tool calling | `q4_K_M` ou `q5_K_M` | Sempre residente; baixa latência |
| Codificação agêntica | Coder 30B-classe (ex. MoE 30B com ~3B ativos) | `q4_K_M` | Contexto longo p/ repositório; cabe folgado em 32 GB |
| Raciocínio complexo | Modelo de reasoning 27–34B | `q4_K_M` | Para etapas que exigem cadeia de raciocínio |
| Uso geral / redação | 13–27B equilibrado | `q5_K_M` | |

> Os nomes e versões exatos de modelos mudam rápido. Consulte `ollama.com/library` para os lançamentos atuais e prefira sempre a tag mais recente da família que você escolher. O importante aqui é a **estratégia de papéis**, não o nome específico do modelo.

### Baixe os modelos

**Comandos:**
```bash
ollama pull <modelo-roteador>
ollama pull <modelo-coder>
ollama pull <modelo-reasoning>
```

**Problemas encontrados:** nenhum até o momento.

## 4.2 Modelfiles — configuração por papel

Um **Modelfile** é uma receita que cria uma versão nomeada de um modelo com parâmetros fixos (contexto, offload, etc.). Isso é perfeito para o padrão agêntico: cada papel ganha sua configuração ideal.

### Modelfile do roteador (contexto pequeno, sempre na GPU)

Crie um arquivo chamado `Modelfile-router`:

```dockerfile
FROM <modelo-roteador>
PARAMETER num_ctx 4096
PARAMETER num_gpu 99
PARAMETER temperature 0.3
SYSTEM """Você é um roteador de tarefas. Analise o pedido e decida qual
modelo especialista deve executá-lo, respondendo de forma concisa."""
```

**Por quê cada linha:**
- `num_ctx 4096`: o roteador só precisa ver o pedido atual e decidir — contexto pequeno economiza VRAM (KV cache menor), deixando espaço para o especialista.
- `num_gpu 99`: força offload máximo — como é um modelo pequeno, ele cabe 100% na GPU tranquilamente.
- `temperature 0.3`: decisões de roteamento devem ser determinísticas e previsíveis, não criativas.

### Modelfile do coder (contexto grande)

Crie `Modelfile-coder`:

```dockerfile
FROM <modelo-coder>
PARAMETER num_ctx 32768
PARAMETER num_gpu 99
PARAMETER temperature 0.2
```

**Por quê:** tarefas de código agêntico acumulam muito contexto (arquivos, histórico de ferramentas), então um `num_ctx` generoso faz sentido — desde que sobre VRAM após carregar os pesos. `num_gpu 99` mantém tudo na GPU. Temperatura baixa para código mais determinístico.

### Construa os modelos nomeados

**Comandos:**
```bash
ollama create router -f ./Modelfile-router
ollama create coder -f ./Modelfile-coder
```

**Saída / observações:** agora você pode chamar `router` e `coder` como modelos próprios, cada um com sua configuração embutida.

**Problemas encontrados:** nenhum até o momento.

### Controle de residência fino via API

Além do `OLLAMA_KEEP_ALIVE` global (veja [03-configuracao.md](03-configuracao.md)), você controla a permanência de cada modelo **por chamada**, o que é ideal no fluxo agêntico — seu orquestrador sabe melhor que o Ollama o que vem a seguir:

```bash
# Roteador: mantém residente indefinidamente (-1)
curl http://localhost:11434/api/generate -d '{
  "model": "router",
  "prompt": "...",
  "keep_alive": -1
}'

# Especialista: libera rápido após uso pontual
curl http://localhost:11434/api/generate -d '{
  "model": "coder",
  "prompt": "...",
  "keep_alive": "2m"
}'
```

**Por quê:** manter o roteador sempre carregado elimina a latência de recarregá-lo a cada decisão; liberar o especialista logo após o uso devolve VRAM para o próximo especialista que a tarefa exigir. Esse controle por-chamada é o que torna o multi-modelo fluido numa única GPU.
