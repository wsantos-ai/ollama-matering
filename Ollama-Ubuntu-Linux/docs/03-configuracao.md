# 3. Configuração

Variáveis de ambiente, diretórios de modelos, ajustes de serviço (systemd), etc.

<!-- As etapas informadas pelo usuário serão adicionadas abaixo, nesta ordem cronológica -->

## Otimização do serviço via variáveis de ambiente

### Criação do override do systemd

**Objetivo:** criar um arquivo de override (drop-in) do serviço `ollama.service` para
adicionar variáveis de ambiente de otimização sem alterar o unit file original instalado
pelo pacote.

**Comandos:**
```bash
sudo systemctl edit ollama
```

**Saída / observações:**
- O comando abre um editor e cria automaticamente o arquivo
  `/etc/systemd/system/ollama.service.d/override.conf`, aplicando as alterações por cima
  do unit file original (que fica intocado).
- As variáveis de ambiente de otimização (ex.: número de threads, contexto, etc.) serão
  adicionadas dentro do bloco `[Service]` desse override na próxima etapa.

**Problemas encontrados:** nenhum até o momento.

### Verificação de hardware antes de definir os valores

**Objetivo:** conhecer a RAM livre e a topologia de CPU (sockets, cores físicos, threads)
da máquina *antes* de decidir os valores das variáveis de otimização — em ambiente
CPU-only esses dois recursos (RAM e CPU) são os fatores limitantes de desempenho.

**Comandos:**
```bash
free -h
nproc
lscpu | grep -E 'Core|Socket|Thread'
```

**Saída / observações:**
```
$ free -h
(a preencher com a saída real)

$ nproc
8

$ lscpu | grep -E 'Core|Socket|Thread'
Thread(s) per core: 1
Core(s) per socket:  1
Socket(s):           8
```
- Total de CPUs lógicas = `Thread(s) per core` × `Core(s) per socket` × `Socket(s)` =
  1 × 1 × 8 = **8**, batendo com o valor retornado por `nproc`.
- Essa topologia (1 thread por core, 1 core por socket, 8 sockets) é típica de **máquina
  virtual/cloud** (KVM/QEMU), onde cada vCPU é exposto ao SO como um socket próprio, em vez
  do padrão de hardware físico (poucos sockets, vários cores/threads cada). Ou seja: esta é
  uma VM com **8 vCPUs**, sem hyperthreading (1 thread/core) e sem GPU.
- Com base nesses 8 vCPUs, `OLLAMA_NUM_THREADS=6` deixa 2 vCPUs de margem para o sistema
  operacional e outros processos, evitando que a inferência sature 100% da CPU disponível.

**Relação entre a saída desses comandos e as variáveis definidas:**
- `lscpu` → `Thread(s) per core` × `Core(s) per socket` × `Socket(s)` = total de threads
  lógicas de CPU disponíveis. `OLLAMA_NUM_THREADS` deve ser definido com base nesse total,
  deixando margem (geralmente 1–2 threads) livre para o sistema operacional e outros
  processos, evitando que a inferência monopolize a CPU e trave o restante do sistema.
- `free -h` (coluna `available`) → RAM efetivamente disponível para uso, já descontando
  cache/buffers reclamáveis. Esse valor orienta:
  - `OLLAMA_MAX_LOADED_MODELS`: quantos modelos cabem simultaneamente na RAM disponível
    (RAM disponível ÷ tamanho aproximado de cada modelo carregado, com folga de
    segurança).
  - `OLLAMA_KEEP_ALIVE`: manter um modelo carregado por mais tempo (ex.: 60m) só compensa
    se houver RAM sobrando para isso sem competir com outros modelos/processos; em
    máquinas com pouca RAM livre, um valor mais curto evita pressão de memória.
- Em resumo: `OLLAMA_NUM_THREADS` é dimensionado pela saída do `lscpu`, e
  `OLLAMA_MAX_LOADED_MODELS`/`OLLAMA_KEEP_ALIVE` são dimensionados pela saída do `free -h`.

**Problemas encontrados:** nenhum até o momento.

### Variáveis de ambiente adicionadas ao override

**Objetivo:** ajustar rede, uso de CPU e diagnóstico do Ollama para o ambiente sem GPU.

**Conteúdo de `/etc/systemd/system/ollama.service.d/override.conf`:**
```ini
[Service]
# --- Rede ---
Environment="OLLAMA_HOST=0.0.0.0:11434"

# --- Otimização de CPU ---
Environment="OLLAMA_NUM_THREADS=6"
Environment="OLLAMA_NUM_PARALLEL=1"
Environment="OLLAMA_MAX_LOADED_MODELS=2"
Environment="OLLAMA_KEEP_ALIVE=60m"

# --- Limite de CPU via cgroup (ver docs/07-troubleshooting.md) ---
AllowedCPUs=0-5
```

> **Atualização:** o bloco `Environment="OLLAMA_DEBUG=1"` que aparecia aqui foi removido
> depois da fase de validação — ver seção "Remoção do `OLLAMA_DEBUG` após validação"
> mais abaixo. O conteúdo acima já reflete o override sem essa variável.

> **Nota:** `OLLAMA_NUM_THREADS=6` sozinho não se mostrou suficiente para limitar as
> threads usadas pelo Ollama nesta instalação — o processo continuava usando as 8 vCPUs.
> Quem garante o limite de fato é o `AllowedCPUs=0-5`, adicionado depois. Detalhes do
> diagnóstico e da correção em
> [docs/07-troubleshooting.md](07-troubleshooting.md).

**Saída / observações:**
- `OLLAMA_HOST=0.0.0.0:11434` expõe a API do Ollama em todas as interfaces de rede, não
  só em `localhost` — necessário para acessar o serviço a partir de outras máquinas da
  rede local. **Atenção:** isso expõe a API sem autenticação; considerar firewall
  (ufw/iptables) restringindo a porta 11434 às origens confiáveis.
- `OLLAMA_NUM_THREADS=6` limita o número de threads de CPU usadas na inferência — valor
  definido com base na saída de `lscpu` (ver seção "Verificação de hardware" acima),
  deixando margem para o sistema operacional.
- `OLLAMA_NUM_PARALLEL=1` desabilita processamento paralelo de requisições simultâneas,
  evitando concorrência por CPU entre múltiplas chamadas (importante em ambiente sem GPU,
  onde CPU é o recurso limitante).
- `OLLAMA_MAX_LOADED_MODELS=2` limita a quantidade de modelos mantidos carregados em
  memória ao mesmo tempo, controlando o uso de RAM.
- `OLLAMA_KEEP_ALIVE=60m` mantém o modelo carregado em memória por 60 minutos após o
  último uso, evitando recarregar o modelo do disco a cada requisição.
- `OLLAMA_DEBUG=1` ativou logs verbosos durante a fase de configuração/testes (removido
  depois — ver seção seguinte).

**Problemas encontrados:** nenhum até o momento.

### Remoção do `OLLAMA_DEBUG` após validação

**Objetivo:** desligar os logs em modo debug depois de validar a configuração (rede,
threads, `AllowedCPUs`, keep-alive), já que logs verbosos têm custo de performance e de
espaço em disco — conforme a própria recomendação deixada na etapa anterior.

**Comandos:**
```bash
sudo systemctl edit ollama   # remover a linha Environment="OLLAMA_DEBUG=1"
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

**Saída / observações:**
- Validação feita previamente com `journalctl -u ollama -f` e
  `journalctl -u ollama --since "1 hour ago"` (ver
  [docs/05-testes-validacao.md](05-testes-validacao.md)), confirmando o comportamento
  esperado do serviço (incluindo a correção do `AllowedCPUs`, ver
  [docs/07-troubleshooting.md](07-troubleshooting.md)) antes de desligar o modo debug.
- Após a remoção, o override do serviço ficou apenas com as variáveis de rede, otimização
  de CPU/RAM e o limite de `AllowedCPUs` (ver conteúdo atualizado do `override.conf` na
  seção anterior).

**Problemas encontrados:** nenhum.

### Aplicação do override

**Objetivo:** recarregar a configuração do systemd e reiniciar o serviço para que as novas
variáveis de ambiente entrem em vigor.

**Comandos:**
```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
systemctl status ollama
```

- `daemon-reload`: faz o systemd reler os unit files/overrides do disco (não reinicia o
  serviço sozinho — só atualiza a config em memória).
- `restart`: efetivamente reinicia o processo do Ollama já com as novas variáveis de
  ambiente aplicadas.

**Saída / observações:**
- `systemctl status ollama` confirmou o serviço `active (running)` após o restart, ou
  seja, o override foi aplicado sem erros e as variáveis de ambiente (rede, threads,
  paralelismo, modelos carregados, keep-alive e debug) já estão em vigor no serviço.

**Problemas encontrados:** nenhum.

## Firewall (UFW)

### Liberação da porta SSH (22/tcp)

**Objetivo:** garantir acesso SSH à máquina antes de habilitar o UFW, evitando ficar
trancado para fora do servidor ao ativar o firewall. Relacionado ao ponto levantado na
seção de variáveis: como `OLLAMA_HOST=0.0.0.0:11434` expõe a API do Ollama em todas as
interfaces, o UFW é o mecanismo usado para restringir quais origens podem alcançar essa
porta.

**Comandos:**
```bash
sudo ufw allow 22/tcp
```

**Saída / observações:**
```
Rules updated
Rules updated (v6)
```
- As duas linhas confirmam que a regra foi adicionada tanto para IPv4 quanto para IPv6.
- Nesta etapa o UFW ainda não foi habilitado (`ufw enable`) — a regra fica registrada e
  só passa a ser aplicada quando o firewall for ativado. É importante liberar a porta 22
  *antes* de habilitar o UFW justamente para não perder o acesso remoto via SSH.
**Problemas encontrados:** nenhum até o momento.

### Liberação restrita da porta da API do Ollama (11434/tcp)

**Objetivo:** permitir acesso à API do Ollama (exposta em todas as interfaces via
`OLLAMA_HOST=0.0.0.0:11434`) apenas a partir da rede local confiável, em vez de deixá-la
aberta para qualquer origem — mitigando o risco apontado anteriormente de a API não ter
autenticação.

**Comandos:**
```bash
sudo ufw allow from 192.168.1.0/24 to any port 11434 proto tcp
```

**Saída / observações:**
- A regra restringe o acesso à porta 11434/tcp apenas a origens dentro da faixa
  `192.168.1.0/24` (rede local), diferente da regra da porta 22, que ficou liberada para
  qualquer origem — aqui a intenção é já nascer restrita, sem depender de UFW `enable`
  para ser útil como controle de acesso.
**Problemas encontrados:** nenhum até o momento.

### Habilitação do UFW

**Objetivo:** ativar o firewall com as regras já definidas (22/tcp liberado geral,
11434/tcp liberado apenas para a rede local).

**Comandos:**
```bash
sudo ufw enable
```

**Saída / observações:**
```
Firewall is active and enabled on system startup
```
- Confirma que o UFW passou a filtrar o tráfego imediatamente e que essa configuração
  persiste após reinicializações da máquina.
- Como a porta 22 já havia sido liberada antes deste passo, o acesso SSH não foi
  interrompido pela ativação do firewall.

**Problemas encontrados:** nenhum.

### Verificação das regras ativas (`ufw status`)

**Objetivo:** conferir que as regras de firewall configuradas estão de fato ativas e
corretas após habilitar o UFW.

**Comandos:**
```bash
sudo ufw status
```

**Saída / observações:**
```
Status: active

To          Action          From
--          ------          ----
22/tcp      ALLOW           Anywhere
11434/tcp   ALLOW           192.168.1.0/24
22/tcp (v6) ALLOW           Anywhere (v6)
```
- Confirma o esperado: SSH (22/tcp) liberado para qualquer origem (IPv4 e IPv6) e a API
  do Ollama (11434/tcp) restrita apenas à rede local `192.168.1.0/24`.
- Não há regra IPv6 equivalente para a porta 11434 — na prática, a API do Ollama fica
  inacessível via IPv6, o que não é um problema neste ambiente (rede local IPv4), mas
  vale ter em mente caso a rede passe a usar IPv6 no futuro.

**Problemas encontrados:** nenhum.
