# 1. Pré-requisitos e ambiente

Conceitos fundamentais para entender antes de configurar qualquer coisa, ficha técnica da
GPU Tesla V100S e preparação do sistema operacional (Oracle Linux Server 9.8) antes de
instalar drivers e o Ollama.

## 1.1 Como um LLM realmente roda numa GPU

Antes de otimizar, você precisa de um modelo mental correto do que acontece quando você digita `ollama run`.

### O modelo é um monte de "pesos" (weights)

Um LLM é, na prática, um conjunto gigante de números (os *pesos*) organizados em **camadas** (layers). Um modelo de 7 bilhões de parâmetros tem 7 bilhões desses números. Para gerar cada palavra (token), a GPU faz esses números passarem por multiplicações de matrizes, camada por camada.

**Onde ficam esses pesos importa muito:**

- Se todos os pesos estão na **VRAM** (memória da GPU), a inferência é rápida — a GPU tem acesso direto e banda altíssima.
- Se parte dos pesos fica na **RAM do sistema** (memória comum), a GPU precisa buscar dados pela ponte PCIe, que é ordens de magnitude mais lenta.

Esse é o conceito central de toda otimização: **manter o máximo possível do modelo dentro da VRAM.**

### Quantização: encolhendo o modelo sem perder muito

Cada peso, por padrão, ocuparia 16 ou 32 bits. **Quantização** é o processo de representar esses números com menos bits (por exemplo, 4 ou 8), reduzindo drasticamente o tamanho do modelo em memória.

Notações que você verá (formato GGUF, usado pelo Ollama):

| Notação | Bits aprox. | Uso de VRAM | Qualidade |
|---|---|---|---|
| `q8_0` | 8 bits | Alto | Quase idêntica ao original |
| `q5_K_M` | ~5 bits | Médio | Muito boa |
| `q4_K_M` | ~4 bits | Baixo | Boa (padrão recomendado) |
| `q3_K_M` | ~3 bits | Muito baixo | Perceptível perda em tarefas difíceis |

**Por que isso é a alavanca mais importante:** trocar `q8_0` por `q4_K_M` corta o uso de VRAM em aproximadamente 40–50%. Um modelo de 30B que não caberia confortavelmente pode passar a caber com folga, liberando espaço para contexto maior. A perda de qualidade em `q4_K_M` costuma ser pequena o suficiente para a maioria das tarefas.

### Camadas e offload (num_gpu)

Como o modelo é dividido em camadas, o Ollama pode colocar **algumas** camadas na GPU e **outras** na CPU quando o modelo não cabe inteiro na VRAM. Isso se chama *partial offload*.

- Modelo de 32 camadas com `num_gpu=32` → tudo na GPU (ideal).
- `num_gpu=20` → 20 camadas na GPU, 12 na CPU (mais lento).

**A regra de ouro:** um modelo com ~20% das camadas na CPU costuma rodar de 3 a 5 vezes mais lento do que 100% na GPU. Por isso o objetivo prático é sempre chegar a "todas as camadas na GPU". Se não der, o remédio não é aceitar o offload — é escolher uma quantização mais leve ou um modelo menor.

### Contexto e KV cache

O **contexto** (`num_ctx`) é quantos tokens o modelo "lembra" de uma vez — o prompt mais o histórico da conversa. Quanto maior o contexto, mais o modelo consegue considerar de uma vez.

Mas há um custo: para não recalcular tudo a cada token, o modelo guarda cálculos intermediários numa estrutura chamada **KV cache** (key-value cache). Esse cache **também ocupa VRAM**, e cresce proporcionalmente ao tamanho do contexto.

Então a VRAM se divide, na prática, em dois grandes consumidores:

```
VRAM total = pesos do modelo (quantização define)  +  KV cache (contexto define)  +  overhead
```

Aumentar o contexto sem perceber esse custo é uma das causas mais comuns de o modelo "transbordar" para a CPU. Você dimensiona o contexto conforme o que **sobra** de VRAM depois de carregar os pesos.

### Flash Attention

O cálculo de "atenção" (a parte que decide para quais tokens do contexto o modelo presta atenção) é caro em memória no método tradicional, porque materializa matrizes intermediárias enormes.

**Flash Attention** é uma reorganização desse cálculo que evita materializar essas matrizes, reduzindo o uso de memória e o tráfego entre a memória principal da GPU e seu cache rápido. O resultado:

- Menos VRAM usada pela atenção → sobra mais para contexto ou camadas.
- Inferência mais rápida, sobretudo em **contextos longos**.

É especialmente valioso em workflows agênticos e RAG, onde o contexto cresce rápido com histórico de ferramentas. Guarde este conceito — ele tem uma pegadinha específica na V100 (veja abaixo).

## 1.2 Conhecendo a Tesla V100S (e por que ela é especial)

Otimizar sem entender o hardware é chutar. A V100S tem características que mudam decisões concretas de configuração.

### Ficha técnica que importa

- **Arquitetura Volta**, **Compute Capability 7.0** (SM70). É uma placa de data center poderosa, porém de uma geração mais antiga.
- **32 GB de memória HBM2** com banda altíssima (~1.100 GB/s). Muito espaço e muita velocidade de memória — ótimo para LLMs.
- Suporta CUDA 12 normalmente (CUDA 12 cobre compute capability 5.0 e superiores).

### As duas limitações que você precisa memorizar

**(1) Sem BFloat16 nativo.** O tipo de dado `bfloat16` só é suportado em GPUs com compute capability 8.0 ou superior (Ampere em diante). A V100 (7.0) precisa usar `float16` (fp16) no lugar. Na prática, com Ollama isso raramente te bloqueia — os modelos GGUF quantizados já lidam com isso —, mas é a razão pela qual alguns frameworks (como vLLM) exigem que você passe `--dtype half` explicitamente na V100. **Fique atento a mensagens de erro mencionando bfloat16.**

**(2) Flash Attention é suporte parcial.** O FlashAttention 2/3 "oficial" exige arquitetura Ampere ou mais nova — a Volta não é suportada por ele. O Ollama/llama.cpp tem uma implementação própria e mais simples que *funciona parcialmente* na V100, mas de forma inconsistente entre famílias de modelo: em testes reais na V100, o mesmo servidor rodou alguns modelos com Flash Attention ativo e outros com ele desabilitado automaticamente, mesmo com a variável ligada.

**Consequência prática:** você vai **ligar** `OLLAMA_FLASH_ATTENTION`, mas **não confiar cegamente** — vai verificar nos logs, para cada modelo, se ele realmente ficou `Enabled`. As versões mais recentes do Ollama (a partir da v0.32.0, julho/2026) trouxeram melhorias de Flash Attention justamente para GPUs NVIDIA mais antigas, então manter o Ollama atualizado te ajuda aqui.

### Atenção a bugs de compatibilidade por versão

Há um detalhe importante: algumas versões do Ollama apresentaram o erro `CUDA error: device kernel image is invalid` especificamente em Tesla V100, por incompatibilidade do llama.cpp com a arquitetura Volta mais antiga. Se você topar com esse erro, a saída é atualizar para uma versão corrigida ou, temporariamente, fixar uma versão anterior conhecida por funcionar (veja [07-troubleshooting.md](07-troubleshooting.md)).

### O que tudo isso significa para suas escolhas

Com 32 GB de VRAM você está numa posição confortável:

- Modelos até ~13–20B cabem com quantização generosa (`q5_K_M`, `q8_0`) e contexto grande.
- Modelos de 30–34B cabem bem em `q4_K_M`, ainda sobrando espaço para KV cache.
- Modelos de ~70B são possíveis em quantização agressiva, mas ficam apertados numa placa só — para uso agêntico, ficar na faixa 8–34B é o ponto ideal de latência × qualidade.

## 1.3 Preparando o sistema operacional (Oracle Linux Server 9.8)

> Os comandos abaixo assumem **Oracle Linux Server 9.8** (OL9, `el9`). Para outra minor version da série 9.x o procedimento é o mesmo — apenas confirme a versão antes de prosseguir (passo abaixo). Para OL8, troque `rhel9` por `rhel8` nos URLs de repositório e `el9`/`ol9` por `el8`/`ol8` onde aparecerem. Rode como `root` ou com `sudo`.

### Confirme a versão do sistema operacional

**Objetivo:** confirmar a distribuição e a versão exata do Oracle Linux antes de escolher repositórios e pacotes (que são versionados por major release, `el9`).

**Comandos:**
```bash
cat /etc/os-release
```

**Saída / observações esperadas:**
```
NAME="Oracle Linux Server"
VERSION="9.8"
ID="ol"
ID_LIKE="fedora"
VARIANT="Server"
VARIANT_ID="server"
VERSION_ID="9.8"
PLATFORM_ID="platform:el9"
PRETTY_NAME="Oracle Linux Server 9.8"
```
- `PLATFORM_ID="platform:el9"` é o que importa para os repositórios usados a seguir (CodeReady Builder, EPEL, CUDA `rhel9`) — todos são compatíveis com qualquer minor version `9.x`, então não é necessário trocar nada nos comandos por causa do `9.8` especificamente.

### Atualize o sistema e instale utilitários base

**Comandos:**
```bash
sudo dnf update -y
sudo dnf install -y dnf-plugins-core wget curl tar
```

**Por quê:** o `dnf-plugins-core` fornece o `dnf config-manager`, que usaremos para adicionar os repositórios da NVIDIA. Um sistema atualizado evita conflitos de kernel na hora de compilar o driver.

### Habilite os repositórios necessários (EPEL + CodeReady Builder)

O driver NVIDIA precisa do **DKMS** e de **kernel headers**, que vêm de repositórios que não estão habilitados por padrão no Oracle Linux.

**Comandos:**
```bash
# Habilita o CodeReady Builder (fornece dependências de desenvolvimento)
sudo dnf config-manager --set-enabled ol9_codeready_builder

# Instala o repositório EPEL do Oracle (fornece DKMS e utilitários)
sudo dnf install -y oracle-epel-release-el9
```

**Por quê:** o **DKMS** (Dynamic Kernel Module Support) recompila automaticamente o módulo do driver toda vez que o kernel do Linux é atualizado. Sem ele, uma atualização de kernel de rotina quebraria o driver da GPU e você acordaria com o Ollama rodando lento em CPU. É o que garante que sua GPU continue funcionando após updates.

### Instale os headers do kernel em uso

Antes de instalar, confira qual kernel está rodando — o nome do pacote muda dependendo se é UEK ou RHCK:

```bash
uname -r
```

Se a saída terminar em `.el9uek.x86_64` (ex.: `6.12.0-204.92.4.3.1.el9uek.x86_64`), você está no kernel **UEK** (Unbreakable Enterprise Kernel), o padrão do Oracle Linux. Se terminar em `.el9.x86_64` (sem `uek`), é o kernel **RHCK** (Red Hat Compatible Kernel).

**Comandos (kernel RHCK — sem `uek` no `uname -r`):**
```bash
sudo dnf install -y kernel-devel-$(uname -r) kernel-headers-$(uname -r) gcc make
```

**Comandos (kernel UEK — com `uek` no `uname -r`, o caso mais comum no Oracle Linux):**
```bash
sudo dnf install -y kernel-uek-devel-$(uname -r) gcc make
```

**Por quê:** o driver é um módulo de kernel; para compilá-lo, o DKMS precisa do pacote `-devel` correspondente **exatamente à versão do kernel que está rodando** (por isso o `$(uname -r)`) — é ele que traz `/usr/src/kernels/$(uname -r)/`, usado pelo DKMS para compilar o módulo. No kernel UEK esse pacote se chama `kernel-uek-devel`, não `kernel-devel` — os nomes `kernel-devel`/`kernel-headers` (sem `uek`) pertencem ao kernel RHCK e **não existem** com o sufixo de versão de um kernel UEK, resultando em `No match for argument`. `kernel-uek-headers` normalmente não existe como pacote separado; o `kernel-uek-devel` já é suficiente para o DKMS compilar o driver.

**Problema encontrado:** rodar o comando genérico (`kernel-devel-$(uname -r) kernel-headers-$(uname -r)`) num sistema com kernel UEK ativo (`6.12.0-204.92.4.3.1.el9uek.x86_64`) resultou em:
```
No match for argument: kernel-devel-6.12.0-204.92.4.3.1.el9uek.x86_64
No match for argument: kernel-headers-6.12.0-204.92.4.3.1.el9uek.x86_64
Error: Unable to find a match: kernel-devel-... kernel-headers-...
```
`gcc` e `make` já estavam instalados, então não houve erro neles. **Solução:** usar o comando "kernel UEK" acima (`kernel-uek-devel-$(uname -r)`). Depois de instalar, confirme que o diretório de build existe:
```bash
ls -d /usr/src/kernels/$(uname -r)
```
Se o diretório existir, o DKMS tem o que precisa para compilar o driver NVIDIA no próximo módulo.
