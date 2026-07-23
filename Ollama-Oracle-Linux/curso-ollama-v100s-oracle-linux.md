# Curso Prático: Ambiente Ollama Otimizado em GPU Tesla V100S (Oracle Linux)

> Um guia passo-a-passo, do zero ao ambiente agêntico otimizado, com explicação didática de cada conceito.

---

## Como usar este curso

O curso está dividido em três blocos:

1. **Fundamentos** (Módulos 1–2): os conceitos que você precisa entender *antes* de configurar qualquer coisa. Ler isto primeiro evita 90% dos erros de tuning.
2. **Instalação** (Módulos 3–6): o passo-a-passo prático no Oracle Linux, do driver ao serviço rodando.
3. **Otimização e uso agêntico** (Módulos 7–10): tuning fino, Modelfiles por tarefa, monitoramento e troubleshooting.

Cada seção técnica explica **o quê**, **por quê** e **como**. Não pule os "por quê" — eles são o que diferencia copiar comandos de saber o que está fazendo.

---

# BLOCO 1 — FUNDAMENTOS

## Módulo 1: Como um LLM realmente roda numa GPU

Antes de otimizar, você precisa de um modelo mental correto do que acontece quando você digita `ollama run`.

### 1.1 O modelo é um monte de "pesos" (weights)

Um LLM é, na prática, um conjunto gigante de números (os *pesos*) organizados em **camadas** (layers). Um modelo de 7 bilhões de parâmetros tem 7 bilhões desses números. Para gerar cada palavra (token), a GPU faz esses números passarem por multiplicações de matrizes, camada por camada.

**Onde ficam esses pesos importa muito:**

- Se todos os pesos estão na **VRAM** (memória da GPU), a inferência é rápida — a GPU tem acesso direto e banda altíssima.
- Se parte dos pesos fica na **RAM do sistema** (memória comum), a GPU precisa buscar dados pela ponte PCIe, que é ordens de magnitude mais lenta.

Esse é o conceito central de toda otimização: **manter o máximo possível do modelo dentro da VRAM.**

### 1.2 Quantização: encolhendo o modelo sem perder muito

Cada peso, por padrão, ocuparia 16 ou 32 bits. **Quantização** é o processo de representar esses números com menos bits (por exemplo, 4 ou 8), reduzindo drasticamente o tamanho do modelo em memória.

Notações que você verá (formato GGUF, usado pelo Ollama):

| Notação | Bits aprox. | Uso de VRAM | Qualidade |
|---|---|---|---|
| `q8_0` | 8 bits | Alto | Quase idêntica ao original |
| `q5_K_M` | ~5 bits | Médio | Muito boa |
| `q4_K_M` | ~4 bits | Baixo | Boa (padrão recomendado) |
| `q3_K_M` | ~3 bits | Muito baixo | Perceptível perda em tarefas difíceis |

**Por que isso é a alavanca mais importante:** trocar `q8_0` por `q4_K_M` corta o uso de VRAM em aproximadamente 40–50%. Um modelo de 30B que não caberia confortavelmente pode passar a caber com folga, liberando espaço para contexto maior. A perda de qualidade em `q4_K_M` costuma ser pequena o suficiente para a maioria das tarefas.

### 1.3 Camadas e offload (num_gpu)

Como o modelo é dividido em camadas, o Ollama pode colocar **algumas** camadas na GPU e **outras** na CPU quando o modelo não cabe inteiro na VRAM. Isso se chama *partial offload*.

- Modelo de 32 camadas com `num_gpu=32` → tudo na GPU (ideal).
- `num_gpu=20` → 20 camadas na GPU, 12 na CPU (mais lento).

**A regra de ouro:** um modelo com ~20% das camadas na CPU costuma rodar de 3 a 5 vezes mais lento do que 100% na GPU. Por isso o objetivo prático é sempre chegar a "todas as camadas na GPU". Se não der, o remédio não é aceitar o offload — é escolher uma quantização mais leve ou um modelo menor.

### 1.4 Contexto e KV cache

O **contexto** (`num_ctx`) é quantos tokens o modelo "lembra" de uma vez — o prompt mais o histórico da conversa. Quanto maior o contexto, mais o modelo consegue considerar de uma vez.

Mas há um custo: para não recalcular tudo a cada token, o modelo guarda cálculos intermediários numa estrutura chamada **KV cache** (key-value cache). Esse cache **também ocupa VRAM**, e cresce proporcionalmente ao tamanho do contexto.

Então a VRAM se divide, na prática, em dois grandes consumidores:

```
VRAM total = pesos do modelo (quantização define)  +  KV cache (contexto define)  +  overhead
```

Aumentar o contexto sem perceber esse custo é uma das causas mais comuns de o modelo "transbordar" para a CPU. Você dimensiona o contexto conforme o que **sobra** de VRAM depois de carregar os pesos.

### 1.5 Flash Attention

O cálculo de "atenção" (a parte que decide para quais tokens do contexto o modelo presta atenção) é caro em memória no método tradicional, porque materializa matrizes intermediárias enormes.

**Flash Attention** é uma reorganização desse cálculo que evita materializar essas matrizes, reduzindo o uso de memória e o tráfego entre a memória principal da GPU e seu cache rápido. O resultado:

- Menos VRAM usada pela atenção → sobra mais para contexto ou camadas.
- Inferência mais rápida, sobretudo em **contextos longos**.

É especialmente valioso em workflows agênticos e RAG, onde o contexto cresce rápido com histórico de ferramentas. Guarde este conceito — ele tem uma pegadinha específica na V100 que veremos no Módulo 2.

---

## Módulo 2: Conhecendo a Tesla V100S (e por que ela é especial)

Otimizar sem entender o hardware é chutar. A V100S tem características que mudam decisões concretas de configuração.

### 2.1 Ficha técnica que importa

- **Arquitetura Volta**, **Compute Capability 7.0** (SM70). É uma placa de data center poderosa, porém de uma geração mais antiga.
- **32 GB de memória HBM2** com banda altíssima (~1.100 GB/s). Muito espaço e muita velocidade de memória — ótimo para LLMs.
- Suporta CUDA 12 normalmente (CUDA 12 cobre compute capability 5.0 e superiores).

### 2.2 As duas limitações que você precisa memorizar

**(1) Sem BFloat16 nativo.** O tipo de dado `bfloat16` só é suportado em GPUs com compute capability 8.0 ou superior (Ampere em diante). A V100 (7.0) precisa usar `float16` (fp16) no lugar. Na prática, com Ollama isso raramente te bloqueia — os modelos GGUF quantizados já lidam com isso —, mas é a razão pela qual alguns frameworks (como vLLM) exigem que você passe `--dtype half` explicitamente na V100. **Fique atento a mensagens de erro mencionando bfloat16.**

**(2) Flash Attention é suporte parcial.** O FlashAttention 2/3 "oficial" exige arquitetura Ampere ou mais nova — a Volta não é suportada por ele. O Ollama/llama.cpp tem uma implementação própria e mais simples que *funciona parcialmente* na V100, mas de forma inconsistente entre famílias de modelo: em testes reais na V100, o mesmo servidor rodou alguns modelos com Flash Attention ativo e outros com ele desabilitado automaticamente, mesmo com a variável ligada.

**Consequência prática:** você vai **ligar** `OLLAMA_FLASH_ATTENTION`, mas **não confiar cegamente** — vai verificar nos logs, para cada modelo, se ele realmente ficou `Enabled`. As versões mais recentes do Ollama (a partir da v0.32.0, julho/2026) trouxeram melhorias de Flash Attention justamente para GPUs NVIDIA mais antigas, então manter o Ollama atualizado te ajuda aqui.

### 2.3 Atenção a bugs de compatibilidade por versão

Há um detalhe importante: algumas versões do Ollama apresentaram o erro `CUDA error: device kernel image is invalid` especificamente em Tesla V100, por incompatibilidade do llama.cpp com a arquitetura Volta mais antiga. Se você topar com esse erro, a saída é atualizar para uma versão corrigida ou, temporariamente, fixar uma versão anterior conhecida por funcionar. Trataremos disso no Módulo 10 (Troubleshooting).

### 2.4 O que tudo isso significa para suas escolhas

Com 32 GB de VRAM você está numa posição confortável:

- Modelos até ~13–20B cabem com quantização generosa (`q5_K_M`, `q8_0`) e contexto grande.
- Modelos de 30–34B cabem bem em `q4_K_M`, ainda sobrando espaço para KV cache.
- Modelos de ~70B são possíveis em quantização agressiva, mas ficam apertados numa placa só — para o seu caso agêntico, ficar na faixa 8–34B é o ponto ideal de latência × qualidade.

---

# BLOCO 2 — INSTALAÇÃO NO ORACLE LINUX

> Os comandos abaixo assumem **Oracle Linux 9** (OL9). Para OL8, troque `rhel9` por `rhel8` nos URLs de repositório e `el9`/`ol9` por `el8`/`ol8` onde aparecerem. Rode como `root` ou com `sudo`.

## Módulo 3: Preparando o sistema operacional

### 3.1 Atualize o sistema e instale utilitários base

```bash
sudo dnf update -y
sudo dnf install -y dnf-plugins-core wget curl tar
```

**Por quê:** o `dnf-plugins-core` fornece o `dnf config-manager`, que usaremos para adicionar os repositórios da NVIDIA. Um sistema atualizado evita conflitos de kernel na hora de compilar o driver.

### 3.2 Habilite os repositórios necessários (EPEL + CodeReady Builder)

O driver NVIDIA precisa do **DKMS** e de **kernel headers**, que vêm de repositórios que não estão habilitados por padrão no Oracle Linux.

```bash
# Habilita o CodeReady Builder (fornece dependências de desenvolvimento)
sudo dnf config-manager --set-enabled ol9_codeready_builder

# Instala o repositório EPEL do Oracle (fornece DKMS e utilitários)
sudo dnf install -y oracle-epel-release-el9
```

**Por quê:** o **DKMS** (Dynamic Kernel Module Support) recompila automaticamente o módulo do driver toda vez que o kernel do Linux é atualizado. Sem ele, uma atualização de kernel de rotina quebraria o driver da GPU e você acordaria com o Ollama rodando lento em CPU. É o que garante que sua GPU continue funcionando após updates.

### 3.3 Instale os headers do kernel em uso

```bash
sudo dnf install -y kernel-devel-$(uname -r) kernel-headers-$(uname -r) gcc make
```

**Por quê:** o driver é um módulo de kernel; para compilá-lo, o DKMS precisa dos headers correspondentes **exatamente à versão do kernel que está rodando** (por isso o `$(uname -r)`). Se estes não baterem com o kernel ativo, a compilação falha silenciosamente e a GPU não aparece.

> **Nota sobre o kernel UEK:** o Oracle Linux costuma usar o kernel UEK (Unbreakable Enterprise Kernel) por padrão. Ele funciona com o driver NVIDIA, mas garanta que os `kernel-devel`/`kernel-headers` instalados correspondam ao kernel ativo (UEK ou RHCK). Rode `uname -r` e confira que a versão retornada bate com os pacotes instalados.

---

## Módulo 4: Instalando o driver NVIDIA e o CUDA

### 4.1 Adicione o repositório CUDA da NVIDIA

```bash
sudo dnf config-manager --add-repo \
  https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo

sudo dnf clean expire-cache
```

**Por quê:** esse repositório oficial da NVIDIA fornece tanto o driver quanto o CUDA Toolkit, com atualizações e a integração DKMS já empacotadas. Usar o repositório (em vez de um instalador `.run` avulso) facilita manutenção e updates.

### 4.2 Instale o driver

Para a V100 (Volta, hardware de data center mais antigo), o **driver proprietário** é a escolha mais segura em termos de compatibilidade:

```bash
# Reseta qualquer stream de módulo pré-existente
sudo dnf module reset -y nvidia-driver

# Instala o driver proprietário mais recente + DKMS
sudo dnf module install -y nvidia-driver:latest-dkms
```

**Por quê o proprietário e não o `nvidia-open`:** os módulos de kernel "open" da NVIDIA são plenamente recomendados para arquiteturas mais novas. Na Volta, que é uma geração mais antiga, o driver proprietário tende a oferecer a compatibilidade mais previsível. Se você tiver um motivo específico para usar os módulos open, eles também funcionam na V100 — mas para "instalar e esquecer", o proprietário é o caminho de menor atrito.

### 4.3 Instale o CUDA Toolkit

```bash
sudo dnf install -y cuda-toolkit-12-8
```

**Por quê:** o Ollama traz suas próprias bibliotecas de runtime CUDA compiladas, então tecnicamente ele roda sem o Toolkit completo. Mas ter o Toolkit instalado te dá o `nvidia-smi` sempre acessível, ferramentas de diagnóstico e compatibilidade para outras cargas. Vale o espaço em disco.

### 4.4 Ajuste o PATH

O CUDA instala binários em `/usr/local/cuda/bin`, que não está no PATH padrão.

```bash
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

**Por quê:** sem isso, você pode instalar tudo corretamente e ainda ver `nvidia-smi: command not found`, o que gera pânico desnecessário. Ajustar o PATH resolve.

### 4.5 Reinicie e verifique

```bash
sudo reboot
```

Após reiniciar:

```bash
nvidia-smi
```

**O que você deve ver:** uma tabela listando `Tesla V100S`, a versão do driver, a versão do CUDA e ~32 GB (`32768 MiB`) de memória total. Se a placa aparece aqui, o alicerce está pronto.

**Se não aparecer:** o problema está no driver, não no Ollama. Verifique se os kernel-headers batem com `uname -r`, cheque `dmesg | grep -i nvidia` e confirme que o DKMS compilou o módulo (`dkms status`).

---

## Módulo 5: Instalando o Ollama

### 5.1 Instalação via script oficial

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

**Por quê:** o script oficial detecta a distribuição, instala o binário, cria o usuário de sistema `ollama` e configura um serviço systemd automaticamente. Em ambiente de servidor, rodar como serviço (em vez de `ollama serve` num terminal) é o correto: sobe no boot, reinicia sozinho em caso de falha e roda de forma isolada.

### 5.2 Confirme a instalação

```bash
ollama --version
systemctl status ollama
```

Você deve ver o serviço `active (running)`. Nesse ponto o Ollama já enxerga sua GPU — mas ainda **sem** as otimizações. É o que faremos a seguir.

---

## Módulo 6: Configurando o serviço com as variáveis de otimização

Aqui está o ponto onde a maioria das pessoas erra. **As variáveis de ambiente do Ollama são lidas pelo servidor (`ollama serve`) na inicialização — não pelo comando `ollama run`.** Definir a variável no seu terminal não tem efeito nenhum se o servidor já está rodando como serviço systemd, porque o serviço tem o próprio ambiente. Portanto, configuramos **no systemd**.

### 6.1 Crie um override do serviço

```bash
sudo systemctl edit ollama
```

Isso abre um editor. Adicione o bloco abaixo (dentro das linhas de comentário que o editor mostra):

```ini
[Service]
# --- Rede ---
Environment="OLLAMA_HOST=0.0.0.0:11434"

# --- Otimização de GPU / memória ---
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=f16"
Environment="OLLAMA_MAX_LOADED_MODELS=2"
Environment="OLLAMA_NUM_PARALLEL=1"
Environment="OLLAMA_KEEP_ALIVE=10m"

# --- Diagnóstico (deixe ligado no início, desligue depois) ---
Environment="OLLAMA_DEBUG=1"
```

### 6.2 Entenda cada variável (o "porquê" de cada linha)

**`OLLAMA_HOST=0.0.0.0:11434`**
Faz o Ollama escutar em todas as interfaces de rede, permitindo que seu orquestrador de agentes acesse a API de outra máquina. ⚠️ **Atenção de segurança:** isso expõe a API sem autenticação. Só use em rede confiável e **sempre** com firewall restringindo a porta 11434 à sua sub-rede (veja 6.4). Se o orquestrador roda na mesma máquina do Ollama, deixe `127.0.0.1:11434` (o padrão) e pule o firewall.

**`OLLAMA_FLASH_ATTENTION=1`**
Ativa o suporte a Flash Attention. Como vimos, na V100 o efeito é por-modelo, mas ligar não custa nada e ajuda nos modelos compatíveis, reduzindo VRAM da atenção e acelerando contextos longos — exatamente o cenário agêntico.

**`OLLAMA_KV_CACHE_TYPE=f16`**
Define o tipo de dado do KV cache. Usamos `f16` (float16) porque a V100 **não** tem bfloat16 nativo, e f16 é o formato bem suportado pela Volta. Existem opções mais agressivas (`q8_0`, `q4_0`) que quantizam o próprio cache para economizar ainda mais VRAM — porém a quantização de KV cache no llama.cpp geralmente **depende de Flash Attention ativo**, o que na V100 é inconsistente. Por isso começamos com `f16`, que é o mais seguro; você pode experimentar `q8_0` depois se precisar de mais folga e confirmar que a qualidade se mantém.

**`OLLAMA_MAX_LOADED_MODELS=2`**
Permite manter até 2 modelos carregados em VRAM ao mesmo tempo. Esse é o coração da estratégia agêntica: um modelo pequeno "roteador" fica sempre residente enquanto o modelo pesado da tarefa entra e sai. Com 32 GB você tem espaço para isso.

**`OLLAMA_NUM_PARALLEL=1`**
Quantas requisições simultâneas cada modelo processa. Cada slot paralelo aloca seu próprio KV cache, consumindo VRAM adicional. Numa única GPU, manter em `1` (ou no máximo `2`) deixa o consumo de memória previsível e evita OOM. Deixe seu orquestrador serializar as chamadas pesadas.

**`OLLAMA_KEEP_ALIVE=10m`**
Por quanto tempo um modelo fica na VRAM após a última requisição. O padrão é 5 minutos. Em uso agêntico, subir para 10m evita recarregar do disco a cada troca de tarefa (o carregamento custa segundos e é sentido na latência). No Módulo 8 refinamos isso por-chamada.

**`OLLAMA_DEBUG=1`**
Faz o Ollama logar cada variável resolvida e detalhes de carregamento de cada modelo — inclusive se o Flash Attention ficou `Enabled` ou `Disabled`. **Essencial agora**, no começo, para você validar que tudo pegou. Depois de confirmar, remova para deixar os logs limpos.

### 6.3 Aplique as mudanças

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

**Por quê:** o `daemon-reload` faz o systemd reler a configuração do serviço; o `restart` reinicia o Ollama para que ele leia as novas variáveis (lembre: variáveis só são lidas na inicialização).

### 6.4 (Se usar acesso remoto) Configure o firewall

```bash
# Libera a porta apenas para a sua sub-rede (ajuste o CIDR)
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="11434" protocol="tcp" accept'
sudo firewall-cmd --reload
```

**Por quê:** sem isso, expor `0.0.0.0` deixa qualquer um na rede usar (e abusar) da sua GPU. A regra restringe o acesso à sua sub-rede confiável.

---

# BLOCO 3 — OTIMIZAÇÃO E USO AGÊNTICO

## Módulo 7: Escolhendo os modelos por tarefa

Num ambiente agêntico, você não quer um único modelo genérico — quer o modelo *certo* para cada etapa. A ideia é ter um **roteador leve** que planeja e decide, e **especialistas** que executam.

### 7.1 Um ponto crítico: tool calling

Se seus agentes chamam ferramentas (functions/tools), a qualidade do *tool calling* varia muito entre famílias de modelo. Modelos das famílias Qwen, Llama e Mistral tendem a ter tool calling robusto; alguns modelos (como o Gemma) são notoriamente mais fracos nisso. **Escolha modelos com bom suporte a ferramentas para os papéis que interagem com tools.**

### 7.2 Sugestão de elenco (dentro de 32 GB, faixas de tamanho)

| Papel no agente | Perfil de modelo | Quantização sugerida | Observação |
|---|---|---|---|
| Roteador / planejador | Pequeno, 7–8B, bom em tool calling | `q4_K_M` ou `q5_K_M` | Sempre residente; baixa latência |
| Codificação agêntica | Coder 30B-classe (ex. MoE 30B com ~3B ativos) | `q4_K_M` | Contexto longo p/ repositório; cabe folgado em 32 GB |
| Raciocínio complexo | Modelo de reasoning 27–34B | `q4_K_M` | Para etapas que exigem cadeia de raciocínio |
| Uso geral / redação | 13–27B equilibrado | `q5_K_M` | |

> Os nomes e versões exatos de modelos mudam rápido. Consulte `ollama.com/library` para os lançamentos atuais e prefira sempre a tag mais recente da família que você escolher. O importante aqui é a **estratégia de papéis**, não o nome específico do modelo.

### 7.3 Baixe os modelos

```bash
ollama pull <modelo-roteador>
ollama pull <modelo-coder>
ollama pull <modelo-reasoning>
```

---

## Módulo 8: Modelfiles — configuração por papel

Um **Modelfile** é uma receita que cria uma versão nomeada de um modelo com parâmetros fixos (contexto, offload, etc.). Isso é perfeito para o padrão agêntico: cada papel ganha sua configuração ideal.

### 8.1 Modelfile do roteador (contexto pequeno, sempre na GPU)

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

### 8.2 Modelfile do coder (contexto grande)

Crie `Modelfile-coder`:

```dockerfile
FROM <modelo-coder>
PARAMETER num_ctx 32768
PARAMETER num_gpu 99
PARAMETER temperature 0.2
```

**Por quê:** tarefas de código agêntico acumulam muito contexto (arquivos, histórico de ferramentas), então um `num_ctx` generoso faz sentido — desde que sobre VRAM após carregar os pesos. `num_gpu 99` mantém tudo na GPU. Temperatura baixa para código mais determinístico.

### 8.3 Construa os modelos nomeados

```bash
ollama create router -f ./Modelfile-router
ollama create coder -f ./Modelfile-coder
```

Agora você pode chamar `router` e `coder` como modelos próprios, cada um com sua configuração embutida.

### 8.4 Controle de residência fino via API

Além do `OLLAMA_KEEP_ALIVE` global, você controla a permanência de cada modelo **por chamada**, o que é ideal no fluxo agêntico — seu orquestrador sabe melhor que o Ollama o que vem a seguir:

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

---

## Módulo 9: Verificação e monitoramento

Otimização sem medição é só suposição. Aqui está como confirmar que está tudo funcionando como planejado.

### 9.1 Confirme o que está carregado na VRAM

```bash
ollama ps
```

Mostra quais modelos estão em memória, quanto ocupam, se estão 100% na GPU ou divididos com CPU, e quando serão descarregados. **Se aparecer qualquer coisa diferente de 100% GPU, você tem offload para CPU** — hora de reduzir contexto ou usar quantização mais leve.

### 9.2 Monitore a GPU em tempo real

```bash
watch -n 1 nvidia-smi
```

Acompanhe o uso de VRAM e a utilização do núcleo (`GPU-Util`). Durante a geração, o `GPU-Util` deve subir bem. Se a VRAM estiver perto do limite dos 32 GB, você está no fio da navalha para OOM — deixe uma margem.

### 9.3 Onde consultar os logs

Como o Ollama roda via systemd no Oracle Linux:

```bash
# Todos os logs do serviço
journalctl -u ollama

# Acompanhar em tempo real (ótimo durante testes)
journalctl -u ollama -f

# Só a última hora
journalctl -u ollama --since "1 hour ago"
```

### 9.4 Valide o Flash Attention (o teste da V100)

Com `OLLAMA_DEBUG=1` ativo, carregue um modelo e procure nos logs a linha de carregamento:

```bash
journalctl -u ollama -f | grep -i flash
```

Você verá algo como `FlashAttention:Enabled` ou `FlashAttention:Disabled` no registro de load daquele modelo. **Faça esse teste para cada modelo do seu elenco.** Se ficar `Disabled` num modelo importante, você saberá que não pode contar com a economia de VRAM da atenção para ele — e deve dimensionar o contexto de forma mais conservadora.

### 9.5 Desligue o debug depois

Uma vez validado tudo, edite o serviço (`sudo systemctl edit ollama`), remova a linha `OLLAMA_DEBUG=1`, e reinicie. Logs de debug são verbosos demais para operação contínua.

---

## Módulo 10: Troubleshooting específico da V100

### 10.1 Erro `CUDA error: device kernel image is invalid`

**Causa:** incompatibilidade de certas versões do Ollama/llama.cpp com a arquitetura Volta.
**Solução:**
1. Atualize para a versão mais recente do Ollama (correções para GPUs antigas têm saído continuamente):
   ```bash
   curl -fsSL https://ollama.com/install.sh | sh
   sudo systemctl restart ollama
   ```
2. Se persistir, teste temporariamente uma versão anterior conhecida por funcionar com V100 e acompanhe as issues do repositório do Ollama no GitHub até a correção estabilizar.

### 10.2 Modelo caiu para CPU (lento)

**Sintoma:** `ollama ps` mostra split CPU/GPU; geração lenta.
**Causas e soluções em ordem:**
1. Contexto grande demais → reduza `num_ctx` no Modelfile.
2. Quantização pesada demais → troque `q8_0`/`q5_K_M` por `q4_K_M`.
3. Outro modelo ocupando VRAM → confira `ollama ps` e ajuste `OLLAMA_MAX_LOADED_MODELS` ou o `keep_alive`.

### 10.3 Erro mencionando `bfloat16`

**Causa:** algo tentou usar bf16, que a V100 não suporta nativamente.
**Solução:** com Ollama isso é raro (o GGUF cuida disso), mas se ocorrer com outra ferramenta, force o uso de `float16`/`half`. No Ollama, garanta `OLLAMA_KV_CACHE_TYPE=f16`.

### 10.4 GPU some após suspend/resume (Linux)

**Sintoma:** após o sistema hibernar e voltar, o Ollama não enxerga mais a GPU.
**Solução:** recarregue o módulo UVM da NVIDIA:
```bash
sudo rmmod nvidia_uvm && sudo modprobe nvidia_uvm
```
Em servidor de produção, o ideal é desabilitar suspensão automática para evitar isso.

### 10.5 `nvidia-smi: command not found`

**Causa:** PATH não inclui os binários do CUDA.
**Solução:** revise o passo 4.4 (ajuste de PATH e `LD_LIBRARY_PATH`).

---

## Resumo executivo (checklist final)

1. ✅ Sistema atualizado, EPEL + CodeReady + kernel-headers instalados (DKMS pronto).
2. ✅ Driver NVIDIA proprietário + CUDA Toolkit instalados; `nvidia-smi` mostra a V100S com 32 GB.
3. ✅ Ollama instalado como serviço systemd.
4. ✅ Variáveis configuradas via `systemctl edit ollama`: Flash Attention ligado, KV cache f16, 2 modelos residentes, paralelismo 1, keep-alive 10m.
5. ✅ Elenco de modelos escolhido por papel (roteador leve + especialistas com bom tool calling).
6. ✅ Modelfiles por papel com `num_ctx` e `num_gpu` dimensionados.
7. ✅ Residência controlada por-chamada via API (`keep_alive`).
8. ✅ Validado com `ollama ps`, `nvidia-smi` e logs que os modelos rodam 100% na GPU e o Flash Attention está no estado esperado.
9. ✅ Debug desligado após validação.

### Princípios que você leva deste curso

- **VRAM é o recurso mais escasso.** Toda decisão (quantização, contexto, offload, quantos modelos residentes) é uma disputa por VRAM.
- **100% na GPU é a meta.** Offload para CPU é 3–5x mais lento; prefira quantização mais leve a aceitar offload.
- **Meça, não suponha.** `ollama ps`, `nvidia-smi` e os logs contam a verdade.
- **Conheça seu hardware.** Na V100, bfloat16 não existe e Flash Attention é por-modelo — projete com essas restrições em mente.
- **No fluxo agêntico, o custo é a troca de modelo.** Roteador residente + especialistas sob demanda, com `keep_alive` controlado por chamada, é o padrão que mantém a latência baixa numa única GPU.
