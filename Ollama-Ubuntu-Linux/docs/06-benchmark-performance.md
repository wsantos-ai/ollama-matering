# 6. Benchmark de performance (CPU-only)

Métricas coletadas por modelo: tokens/s, uso de RAM, uso de CPU, latência de primeira
resposta, tempo total. Ambiente sem GPU — todo processamento ocorre em CPU.

<!-- As etapas informadas pelo usuário serão adicionadas abaixo, nesta ordem cronológica -->

### Referência: meça a velocidade real (tokens/s)

**Objetivo:** ter uma referência rápida de como medir e interpretar tokens/s a partir de
qualquer chamada à API do Ollama, sem precisar rodar um teste dedicado toda vez.

Toda resposta da API `/api/generate` ou `/api/chat` traz métricas de performance. A conta
que importa:

```
tokens/s = eval_count / (eval_duration em nanossegundos ÷ 1.000.000.000)
```

- Um alvo saudável para CPU é **10–15 tokens/s**.
- Se estiver bem abaixo disso com um modelo pequeno em `q4_K_M`, revise, nesta ordem:
  1. **Número de threads** — confira se `num_thread`/`OLLAMA_NUM_THREADS` está de fato
     sendo respeitado (ver seção "Script de teste: número ideal de threads" mais abaixo
     neste arquivo, e a Zona 1 do [monitoramento.md](../../monitoramento.md) — barras de CPU
     verdes).
  2. **Swap acontecendo** — verificar a barra `Swp` no `htop` (ver
     [monitoramento.md](../../monitoramento.md)); swap ativo derruba drasticamente o
     tokens/s porque troca RAM por disco.
  3. **Contexto grande demais** — um `num_ctx` alto (como o `8192` do modelo
     `llama3b-cpu`, ver [docs/04-modelos-download-execucao.md](04-modelos-download-execucao.md))
     aumenta o KV cache e pode reduzir o throughput; testar com um contexto menor ajuda a
     isolar se o gargalo é o contexto ou outro fator.
- Os testes registrados abaixo neste arquivo (ex.: ~4,90 tokens/s para o
  `llama3.1:8b-instruct-q4_K_M` e ~10,10 tokens/s para o `llama3.2:3b`) servem de linha de
  base para comparar com esse alvo de 10–15 tokens/s neste ambiente.

### Teste de desempenho via API (`eval_count` / `eval_duration`)

**Objetivo:** medir a velocidade de geração de tokens do modelo `llama3.1:8b-instruct-q4_K_M`
chamando diretamente a API HTTP do Ollama, extraindo apenas os campos relevantes de
performance da resposta.

**Comandos:**
```bash
curl -s http://localhost:11434/api/generate -d '{
  "model": "llama3.1:8b-instruct-q4_K_M",
  "prompt": "Explique fotossíntese em um parágrafo.",
  "options": { "num_thread": 6 },
  "stream": false
}' | grep -o '"eval_count":[0-9]*\|"eval_duration":[0-9]*'
```

**Saída / observações:**
- `stream: false` faz a API retornar a resposta completa em um único JSON (em vez de
  streaming token a token), facilitando extrair métricas de uma vez com `grep`.
- `options.num_thread: 6` força essa chamada específica a usar 6 threads, alinhado ao
  `OLLAMA_NUM_THREADS=6` já definido no override do serviço (ver
  [docs/03-configuracao.md](03-configuracao.md)).
- Campos extraídos da resposta da API:
  - `eval_count`: quantidade de tokens gerados na resposta (não inclui o processamento do
    prompt de entrada, que é contado separadamente em `prompt_eval_count`).
  - `eval_duration`: tempo gasto gerando esses tokens, em **nanossegundos**.
- Cálculo de throughput (tokens/segundo):
  ```
  tokens_por_segundo = eval_count / (eval_duration / 1_000_000_000)
  ```
- `llama3.1:8b-instruct-q4_K_M` é uma versão quantizada em 4 bits (Q4_K_M) do modelo 8B —
  a quantização reduz uso de RAM e acelera a inferência em CPU, com uma perda de qualidade
  geralmente pequena, sendo uma escolha comum para rodar modelos maiores sem GPU.

**Resultado (`eval_count` / `eval_duration`):**
```
"eval_count": 64
"eval_duration": 13061666359
```
- `eval_duration` em segundos: 13.061666359 s
- Tokens/s = 64 / 13.061666359 ≈ **4,90 tokens/s**

**Problemas encontrados:** nenhum até o momento.

### Teste de desempenho — llama3.2:3b (comparativo)

**Objetivo:** rodar o mesmo teste de performance no modelo `llama3.2:3b` (já usado na etapa
de smoke test, ver [docs/04-modelos-download-execucao.md](04-modelos-download-execucao.md))
para comparar throughput com o `llama3.1:8b-instruct-q4_K_M`.

**Comandos:**
```bash
curl -s http://localhost:11434/api/generate -d '{
  "model": "llama3.2:3b",
  "prompt": "Explique fotossíntese em um parágrafo.",
  "options": { "num_thread": 6 },
  "stream": false
}' | grep -o '"eval_count":[0-9]*\|"eval_duration":[0-9]*'
```

**Resultado (`eval_count` / `eval_duration`):**
```
"eval_count": 138
"eval_duration": 13656859802
```
- `eval_duration` em segundos: 13.656859802 s
- Tokens/s = 138 / 13.656859802 ≈ **10,10 tokens/s**

**Problemas encontrados:** nenhum até o momento.

### Comparativo e justificativa do modelo mais adequado ao ambiente

| Modelo | Parâmetros | Quantização | eval_count | eval_duration | Tokens/s |
|---|---|---|---|---|---|
| llama3.1:8b-instruct-q4_K_M | 8B | Q4_K_M (4-bit) | 64 | 13,06 s | ~4,90 |
| llama3.2:3b | 3B | padrão (do model registry) | 138 | 13,66 s | ~10,10 |

**Análise:**
- O `llama3.2:3b` foi **~2,06x mais rápido** em tokens/s do que o `llama3.1:8b-instruct-q4_K_M`
  (10,10 vs 4,90 tokens/s), mesmo o modelo 8B estando quantizado em 4 bits — a diferença de
  tamanho de parâmetros (3B vs 8B) pesa mais no throughput em CPU do que o ganho trazido
  pela quantização do modelo maior.
- Em VM sem GPU (8 vCPUs, sem hyperthreading — ver
  [docs/03-configuracao.md](03-configuracao.md)), o custo de inferência é 100% CPU-bound,
  então o número de parâmetros ativos por token processado domina o tempo de resposta.
- **Recomendação para este ambiente:**
  - Para casos de uso **interativos** (chat, respostas rápidas, throughput como prioridade),
    `llama3.2:3b` é a escolha mais adequada — quase o dobro da velocidade, com consumo de
    RAM menor (permitindo, inclusive, manter mais modelos carregados simultaneamente dentro
    do limite de `OLLAMA_MAX_LOADED_MODELS=2`).
  - `llama3.1:8b-instruct-q4_K_M` continua sendo válido para tarefas onde a **qualidade/
    capacidade de raciocínio** do modelo maior compensa a latência mais alta (ex.: tarefas
    em lote, sem interação em tempo real, ou onde a resposta do 3B não é satisfatória).
  - Regra prática para CPU-only: preferir o menor modelo que atenda à qualidade exigida pela
    tarefa, em vez do modelo mais "capaz" disponível — em GPU essa equação muda, mas em CPU o
    número de parâmetros é o fator que mais penaliza o tempo de resposta.

### Script de teste: número ideal de threads (`num_thread`)

**Objetivo:** variar `num_thread` (4 a 8) no modelo `llama3.2:3b`, medindo a média de
tokens/s em 3 execuções por valor, para decidir empiricamente o melhor valor de
`OLLAMA_NUM_THREADS`/`num_thread` neste ambiente (VM de 8 vCPUs, sem hyperthreading — ver
[docs/03-configuracao.md](03-configuracao.md)).

**Script (`benchmark-threads.sh`):**
```bash
#!/bin/bash
MODELO="llama3.2:3b"
PROMPT="Explique o processo de fotossíntese em detalhes."
REPETICOES=3

echo "Modelo: $MODELO"
echo "threads | tokens/s (média de $REPETICOES execuções)"
echo "--------|----------------------------------"

for t in 4 5 6 7 8; do
  soma=0
  for i in $(seq 1 $REPETICOES); do
    resultado=$(curl -s http://localhost:11434/api/generate -d "{
      \"model\": \"$MODELO\",
      \"prompt\": \"$PROMPT\",
      \"options\": { \"num_thread\": $t },
      \"stream\": false
    }")
    count=$(echo "$resultado" | grep -o '"eval_count":[0-9]*' | cut -d: -f2)
    dur=$(echo "$resultado" | grep -o '"eval_duration":[0-9]*' | cut -d: -f2)
    tps=$(echo "scale=2; $count / ($dur / 1000000000)" | bc)
    soma=$(echo "$soma + $tps" | bc)
  done
  media=$(echo "scale=2; $soma / $REPETICOES" | bc)
  printf "   %2d   | %s\n" "$t" "$media"
done
```
- O script automatiza exatamente o cálculo manual feito no teste anterior
  (`eval_count` / `eval_duration` → tokens/s), repetindo 3x por valor de thread para
  suavizar variação entre execuções, e varrendo de 4 até 8 (o total de vCPUs da máquina).

**Resultado:**

| Threads | Tokens/s (média de 3 execuções) |
|---|---|
| 4 | 5,51 |
| 5 | 6,20 |
| 6 | 7,79 |
| 7 | 8,20 |
| 8 | 8,92 |

**Análise:**
- O throughput cresce de forma consistente com o número de threads, do mínimo testado (4)
  até o máximo de vCPUs da máquina (8) — esperado, já que cada thread adicional processa
  mais trabalho em paralelo em uma CPU sem GPU.
- O ganho **não é linear**: de 4→5 threads o ganho é de ~12,5%, de 5→6 é o maior salto
  (~25,6%), e de 6→7 e 7→8 os ganhos caem para ~5,3% e ~8,8% respectivamente — indicando
  retornos decrescentes à medida que se aproxima do total de vCPUs disponíveis (8).
- Usar `num_thread=8` (100% das vCPUs) maximiza o throughput bruto (8,92 tokens/s), mas
  não deixa **nenhuma margem de CPU** para o sistema operacional e outros processos (SSH,
  UFW, systemd, etc.) — risco de picos de latência ou de a máquina ficar pouco responsiva
  sob carga simultânea.
- `num_thread=7` entrega 8,20 tokens/s — **~92% do throughput máximo obtido com 8** —
  deixando 1 vCPU livre para o sistema, um bom equilíbrio entre desempenho e estabilidade.
- `num_thread=6` (valor atualmente configurado em `OLLAMA_NUM_THREADS` e no Modelfile
  `llama3b-cpu`) entrega 7,79 tokens/s, ~87% do máximo, com 2 vCPUs de margem — a opção
  mais conservadora quanto à responsividade do sistema.

**Recomendação:**
- Para este ambiente (VM compartilhada, rodando também SSH, firewall e demais serviços do
  SO), **manter `num_thread=6`** é uma escolha justificada: abre mão de ~13% de throughput
  em troca de uma margem confortável de CPU para o restante do sistema.
- Caso a máquina seja dedicada exclusivamente à inferência (sem outros serviços
  concorrendo por CPU), **`num_thread=7`** é o melhor custo-benefício — quase o throughput
  máximo, ainda com 1 vCPU de folga.
- Usar `num_thread=8` só se throughput bruto for a prioridade absoluta e não houver
  preocupação com a responsividade do restante do sistema.

**Problemas encontrados:** nenhum até o momento.
