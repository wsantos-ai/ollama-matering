# RAM sobrando x desempenho: por que não são a mesma coisa

Este guia é genérico — vale para qualquer ambiente (bare-metal, VM ou cloud) onde você rode LLMs localmente via Ollama (ou similar) em CPU, sempre que a dúvida for "tenho RAM livre, vale subir para um modelo maior?".

## Por que a memória não encher não significa folga de desempenho

Lembre da analogia do corredor estreito. A RAM é a despensa: ela guarda o livro de receitas. Ter uma despensa grande e meio vazia não faz o cozinheiro andar mais rápido — quem dita a velocidade é a largura do corredor (banda de memória) e o número de cozinheiros (os núcleos de CPU disponíveis).

Ou seja: a RAM sobrando é espaço de armazenamento ocioso, não capacidade de processamento ociosa. São recursos diferentes. O gargalo continua sendo CPU e banda de memória, que provavelmente já estão no talo durante a inferência.

## O custo real de subir de modelo

Aqui está a parte concreta. Como cada token gerado exige ler o modelo inteiro da memória, a velocidade cai de forma aproximadamente proporcional ao tamanho do modelo.

**Exemplo** (máquina com 54 GB de RAM disponível, partindo de um 8B em q4_K_M ~5 GB):

| Modelo (q4_K_M) | RAM aprox. | Velocidade estimada | Cabe nos 54 GB (exemplo)? |
|---|---|---|---|
| 8B (atual) | ~5 GB | ~10 tok/s | Sim, sobrando |
| 14B | ~9 GB | ~6 tok/s | Sim |
| 32B | ~20 GB | ~2,5 tok/s | Sim |
| 70B | ~40 GB | ~1,2 tok/s | Sim, apertado |

Repare: todos cabem nessa RAM de exemplo. Até o 70B entraria confortavelmente — e geraria cerca de uma palavra por segundo. Cabe, mas não serve. O mesmo raciocínio vale proporcionalmente para qualquer quantidade de RAM disponível: caber na memória nunca é garantia de velocidade utilizável.

Uma referência prática para decidir, independente do ambiente:
- **abaixo de 5 tokens/s** o uso interativo começa a ficar penoso (você espera visivelmente cada frase);
- **abaixo de 2 tokens/s**, só faz sentido para tarefas em lote que você deixa rodando e volta depois.

## O que realmente vale fazer com RAM livre

Em vez de gastá-la num modelo maior, use-a nas coisas que não custam velocidade:

1. **Contexto maior.** Aqui sim há ganho real e barato. Suba o `num_ctx` para 16384 ou mais — o KV cache tem espaço de sobra, e isso permite conversas longas e documentos maiores sem truncar.
2. **Melhor quantização do mesmo modelo.** Subir de q4_K_M para q5_K_M no mesmo modelo custa uns 25% de velocidade, mas melhora a qualidade das respostas. É um trade-off muito mais favorável do que dobrar o tamanho do modelo.
3. **Manter modelos carregados.** Com `OLLAMA_MAX_LOADED_MODELS` ajustado para o número de modelos que você alterna com frequência (ex.: 2) e keep-alive alto, você elimina os cold starts entre trocas — ganho direto de latência percebida, sem custo de velocidade de geração.
4. **Cache de disco do sistema.** A RAM livre que o Linux usa como cache faz o carregamento inicial dos modelos ser bem mais rápido nas vezes seguintes. Isso já acontece automaticamente, sem configuração extra.

## O experimento que vale fazer

Se ainda quiser explorar um modelo maior, o passo sensato é um degrau só: teste um modelo um nível acima (ex.: 13–14B se hoje você usa 7–8B) em q4_K_M e meça os tokens/s.

```bash
ollama pull <modelo-14b>-q4_K_M
```

Se der 6+ tok/s e a qualidade for notavelmente melhor para as suas tarefas, pode valer a troca. Se cair para 3–4 tok/s, você vai perceber a diferença no dia a dia e provavelmente vai querer voltar.

**Decida pelo número de tokens/s medido, nunca pela RAM disponível.** Essa é a regra que evita a armadilha.

## Como conferir o consumo real (e não se enganar)

Ao olhar o monitor de recursos (ex.: `htop`), confira o valor `RES` especificamente do processo do Ollama, e não a barra `Mem` geral. Boa parte do que aparece preenchido nessa barra costuma ser cache do sistema, que não é consumo do modelo — o `RES` dá o número honesto de quanto o modelo realmente ocupa. Ver [monitoramento.md](monitoramento.md) para o guia completo de como ler cada zona do `htop` durante um benchmark.
