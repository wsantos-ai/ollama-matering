# Monitoramento com htop

O `htop` parece assustador no começo, mas ele tem só **três zonas**. Depois que você identifica cada uma, fica fácil. Este guia é genérico — vale para qualquer distribuição Linux e qualquer ambiente (bare-metal, VM ou cloud) onde você esteja rodando o Ollama em CPU.

## Instalação e abertura

Se ainda não estiver instalado, use o gerenciador de pacotes da sua distribuição, por exemplo:

```bash
# Debian/Ubuntu
sudo apt install -y htop

# RHEL/Oracle Linux/CentOS
sudo dnf install -y htop
```

Para abrir, basta rodar:

```bash
htop
```

## Zona 1: as barrinhas de CPU (canto superior esquerdo)

Você vai ver uma barra para cada núcleo lógico de CPU disponível na máquina. Cada barra mostra o quanto aquele núcleo está trabalhando naquele instante.

As cores dentro da barra não são decorativas, cada uma significa uma coisa:

- **Verde** — trabalho de programas normais (o Ollama gerando texto aparece aqui). É o que você quer ver durante a inferência.
- **Vermelho** — trabalho do próprio sistema operacional (kernel). Um pouco é normal; muito vermelho indica que o sistema está ocupado com disco, rede ou memória.
- **Azul** — processos de baixa prioridade.
- **Ciano (IO-Wait)** — tempo em que o núcleo ficou parado esperando disco. Por padrão o htop costuma misturar isso no vermelho; para ver separado, ative em `F2` (Setup) → `Display options` → `Detailed CPU time`. É a cor a observar no momento de **carregar** um modelo do disco (primeiro `ollama run`, ou quando o tempo de keep-alive expira e o modelo precisa ser lido de novo) — depois que o modelo já está em RAM, essa cor deve sumir durante a geração de tokens.
- **Laranja/cinza claro (Steal)** — em máquina virtual, indica tempo "roubado" pelo hipervisor, ou seja, momentos em que a VM quis processar mas o servidor físico estava ocupado com outro inquilino. Se o ambiente for uma VM/cloud, um steal alto explica quedas de tokens/s sem nenhuma mudança de configuração sua; em bare-metal essa barra deve ficar praticamente zerada.

**O que observar:** ao limitar o número de threads usadas pelo Ollama (por variável de ambiente, parâmetro do modelo ou limite do sistema operacional), o número de barras bem verdes deve bater com esse valor, com o restante dos núcleos quase parados. Se todas as barras ficarem no talo, o limite configurado não está sendo respeitado.

## Zona 2: memória e resumo (canto superior direito)

**Barra `Mem`** — a RAM da máquina. Aqui tem uma pegadinha importante: as cores também têm significados diferentes.

- **Verde** — memória realmente em uso por programas. É o número que importa.
- **Azul e amarelo/laranja** — buffers e cache. **Isso NÃO é memória desperdiçada.** O Linux usa a RAM ociosa para guardar cópias de arquivos e acelerar as coisas, e libera na hora que algum programa precisar. Muita gente se assusta achando que a memória acabou, quando na verdade está só sendo bem aproveitada.

**Barra `Swp`** — o swap. **Esta é a mais crítica ao rodar LLMs.** Swap é quando o Linux desiste e começa a usar o disco como se fosse memória — e disco é milhares de vezes mais lento que RAM. Ao rodar um modelo que caiba na RAM disponível, essa barra deve ficar **em zero**. Se você vir ela subir durante uma inferência, é sinal vermelho: reduza o modelo, a quantização ou o contexto imediatamente.

**`Load average`** — três números, tipo `2.15 1.80 1.42`. Eles mostram a "fila de trabalho" na última 1, 5 e 15 minutos. A regra: **compare com o número de núcleos lógicos da máquina** (mostrado no topo da Zona 1). Abaixo desse número significa que a máquina dá conta; acima significa que tem processo esperando na fila. Os três números juntos mostram tendência — se o primeiro for muito maior que o terceiro, a carga acabou de aumentar.

**`Tasks` e `Uptime`** — quantos processos existem e há quanto tempo a máquina está ligada. Informativos, pouco relevantes para ajuste de performance.

## Zona 3: a lista de processos (a tabela grande)

Cada linha é um programa rodando. As colunas que realmente importam:

- **`RES`** — a coluna mais importante de todas. É a memória **real** que o processo ocupa na RAM. É aqui que você confere quanto o modelo do Ollama está consumindo de verdade. Espere um valor próximo do tamanho do modelo em disco, um pouco maior por causa do KV cache — e esse cache cresce mais quanto maior for o contexto (`num_ctx`) configurado.
- **`VIRT`** — memória "virtual" reservada. **Ignore essa coluna.** Ela mostra números gigantes e assustadores que não correspondem ao uso real. É fonte constante de pânico desnecessário.
- **`CPU%`** — quanto de processador o programa usa. Detalhe crucial: **esse valor pode passar de 100%!** Cada 100% equivale a um núcleo inteiro. Se o Ollama estiver usando vários núcleos a pleno, você verá algo como `600%` — e isso é ótimo, não é erro.
- **`MEM%`** — a mesma informação do `RES`, mas em porcentagem do total.
- **`TIME+`** — tempo acumulado de CPU desde que o processo começou.
- **`Command`** — o nome do programa. Procure por `ollama`.

## Teclas úteis

- **`F6`** — ordena a lista. Aperte e escolha `PERCENT_CPU` ou `PERCENT_MEM` para ver os maiores consumidores no topo.
- **`F4`** — filtra. Digite `ollama` e a tela mostra só os processos dele.
- **`F10`** ou **`q`** — sai.
- **`H`** — esconde threads individuais, deixando a lista bem mais limpa.

## Ritual prático para um benchmark

Deixe o htop aberto numa janela enquanto roda o teste de performance e observe estas coisas, nesta ordem de importância:

1. **A barra `Swp` continua em zero?** Se não, nada mais importa — resolva isso primeiro.
2. **Quantas barras de CPU ficam verdes?** Deve bater com o limite de threads configurado.
3. **Qual o `RES` do processo do Ollama?** Deve ser próximo do tamanho do modelo em disco, mais o KV cache do contexto.
4. **Apareceu ciano (IO-Wait) nas barras de CPU?** Só é esperado no instante em que o modelo é carregado do disco. Se aparecer durante a geração de tokens (com o modelo já supostamente em RAM), é sinal de que o modelo expirou da memória (tempo de keep-alive) e está sendo relido do disco a cada chamada.

**Checagem cruzada:** compare o que o htop mostra com a saída de `ollama ps`, rodada ao mesmo tempo — o `PROCESSOR` do `ollama ps` (ex.: `100% CPU`) e o `CPU%` do processo no htop devem contar a mesma história; se divergirem bastante, algo mudou na configuração entre uma checagem e outra.

Aperte `F4` e digite `ollama` antes de começar — assim você isola exatamente o que quer observar e não se perde no meio de dezenas de linhas.
