# 4. Modelos — download e execução

Comandos de `ollama pull`, `ollama run`, Modelfiles customizados, escolha de modelos
compatíveis com execução apenas em CPU.

<!-- As etapas informadas pelo usuário serão adicionadas abaixo, nesta ordem cronológica -->

### Download e primeira execução do modelo llama3.2:3b

**Objetivo:** baixar e rodar o primeiro modelo para validar o ambiente CPU-only.

**Comandos:**
```bash
ollama run llama3.2:3b
```

**Saída / observações:**
- Como o modelo ainda não estava presente localmente, o comando primeiro faz o `pull`
  (download das camadas do modelo) e, em seguida, abre uma sessão de chat interativa no
  terminal.
- `llama3.2:3b` é um modelo de 3B parâmetros, tamanho adequado para rodar em CPU (menor
  consumo de RAM e inferência mais rápida do que modelos maiores como 8B/13B).

**Problemas encontrados:** nenhum até o momento.

### Criação de um modelo customizado via Modelfile

**Objetivo:** criar uma variante do `llama3.2:3b` com parâmetros de inferência fixados
(threads, contexto e batch), evitando ter que repetir `options` em cada chamada da API e
já alinhada às otimizações de CPU definidas no serviço (ver
[docs/03-configuracao.md](03-configuracao.md)).

**Comandos:**
```bash
mkdir -p ~/ollama-modelfiles
cd ~/ollama-modelfiles

nano Modelfile-llama3b
```

**Conteúdo de `Modelfile-llama3b`:**
```
FROM llama3.2:3b
PARAMETER num_thread 6
PARAMETER num_ctx 8192
PARAMETER num_batch 256
```

**Comandos (criação e verificação do modelo):**
```bash
ollama create llama3b-cpu -f ./Modelfile-llama3b
ollama list
```

**Saída / observações:**
```
NAME                 ID             SIZE     MODIFIED
llama3b-cpu:latest   7a9ac93bdd1b   2.0 GB   9 seconds ago
```
- `FROM llama3.2:3b` usa o modelo já baixado como base, sem precisar reenviar pesos —
  o `ollama create` apenas monta uma nova camada de configuração em cima do modelo
  existente (por isso o processo é rápido e o tamanho reportado, 2.0 GB, é o mesmo do
  modelo base).
- `PARAMETER num_thread 6`: fixa o número de threads de inferência em 6, coerente com o
  `OLLAMA_NUM_THREADS=6` já definido no override do serviço.
- `PARAMETER num_ctx 8192`: define o tamanho da janela de contexto em 8192 tokens — maior
  que o padrão do modelo, permitindo prompts/conversas mais longas, ao custo de mais RAM
  usada por sessão.
- `PARAMETER num_batch 256`: define o tamanho do lote de tokens processados de uma vez
  durante o *prompt processing* — valores maiores tendem a acelerar o processamento do
  prompt de entrada em CPU, à custa de mais uso de RAM/CPU por lote.
- `ollama list` confirmou a criação do modelo customizado `llama3b-cpu:latest`, disponível
  para uso via `ollama run llama3b-cpu` ou pela API, sem precisar repassar essas opções em
  cada requisição.

**Problemas encontrados:** `PARAMETER num_thread 6` não foi suficiente para limitar as
threads usadas pelo Ollama em tempo de execução — o serviço continuou usando as 8 vCPUs
disponíveis. A correção definitiva (limitação via `AllowedCPUs` no systemd) está
documentada em [docs/07-troubleshooting.md](07-troubleshooting.md).
