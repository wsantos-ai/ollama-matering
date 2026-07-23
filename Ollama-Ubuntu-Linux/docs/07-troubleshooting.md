# 7. Troubleshooting

Erros encontrados durante o processo e como foram resolvidos.

<!-- As etapas informadas pelo usuário serão adicionadas abaixo, nesta ordem cronológica -->

## Troubleshooting em CPU (referência geral)

Checklist de diagnóstico para os sintomas mais comuns ao rodar Ollama sem GPU.

### 8.1 Geração muito lenta (abaixo do esperado)

Ataque nas três alavancas de maior impacto, nesta ordem:

1. **Quantização:** está em `q8_0` ou `q5`? Troque para `q4_K_M` (pode dobrar a
   velocidade). Neste projeto, o `llama3.1:8b-instruct-q4_K_M` já usa essa quantização
   (ver [docs/06-benchmark-performance.md](06-benchmark-performance.md)).
2. **Contexto:** reduza `num_ctx` (ex.: de 8192 para 4096 ou 2048). O modelo customizado
   `llama3b-cpu` deste projeto usa `num_ctx 8192` (ver
   [docs/04-modelos-download-execucao.md](04-modelos-download-execucao.md)) — se a
   geração estiver lenta, esse é um dos primeiros parâmetros a testar reduzido.
3. **Keep-alive:** confirme `OLLAMA_KEEP_ALIVE` para evitar cold starts a cada
   requisição. Neste projeto está definido como `60m` (ver
   [docs/03-configuracao.md](03-configuracao.md)); um valor comumente citado como
   referência é `30m` — o valor ideal depende de quanta RAM sobra para manter o modelo
   carregado por mais tempo.

Depois disso, ajuste `num_thread` empiricamente (ver
[docs/06-benchmark-performance.md](06-benchmark-performance.md)) — mas lembre-se de que,
neste ambiente, o limite que efetivamente vale é o `AllowedCPUs` do systemd, não só o
`num_thread`/`OLLAMA_NUM_THREADS` (ver seção acima "`PARAMETER num_thread` do Modelfile
não era respeitado").

### 8.2 O sistema começa a "engasgar" / usar swap

**Causa:** o modelo + KV cache + SO ultrapassaram a RAM física, e o Linux passou a usar
disco como memória (swap) — desastroso para velocidade.

**Solução:** use um modelo menor ou quantização mais leve; reduza `num_ctx`; feche outros
aplicativos. Confirme com `free -h` e `htop` se o swap está sendo usado durante a
inferência (ver [monitoramento.md](../../monitoramento.md), barra `Swp` na Zona 2 — deve
ficar em zero).

### 8.3 `illegal instruction` ou desempenho péssimo

**Causa provável:** CPU sem suporte a AVX2, caindo num modo genérico muito mais lento.

**Solução:** confirme o suporte com:
```bash
grep avx2 /proc/cpuinfo
```
Sem AVX2, o hardware é o limitante — não há tuning (threads, contexto, keep-alive) que
resolva plenamente.

### 8.4 Multi-socket (servidor) rendendo menos que o esperado

**Causa:** em servidores com 2+ sockets físicos, acessar RAM de outro socket é lento
(efeito NUMA).

**Solução:** experimente fixar a execução a um único nó NUMA:
```bash
numactl --cpunodebind=0 --membind=0 ollama serve
```
Ou habilite o modo NUMA do Ollama por chamada (`--numa`). Muitas vezes um único socket
bem alimentado rende mais que dois mal coordenados.

> **Nota para este projeto:** o `lscpu` desta VM reportou `Socket(s): 8` (ver
> [docs/03-configuracao.md](03-configuracao.md)), mas isso **não é NUMA multi-socket
> real** — é a forma como o hipervisor (KVM/QEMU) expõe 8 vCPUs de um único host físico,
> uma por "socket" virtual. Este item (8.4) tende a **não se aplicar** aqui; ele é
> relevante em hardware bare-metal com múltiplos sockets físicos de verdade.

### 8.5 Forçar modo CPU mesmo com uma GPU presente

Se em algum momento houver uma GPU no sistema mas você quiser forçar CPU (para teste ou
comparação):
```bash
CUDA_VISIBLE_DEVICES="" ollama serve
```
Não se aplica a este projeto no momento (ambiente sem GPU), mas fica registrado para
quem for reproduzir esta documentação em uma máquina com GPU e quiser comparar CPU vs GPU.

### `PARAMETER num_thread` do Modelfile não era respeitado (Ollama continuava usando 8 threads)

**Problema:** mesmo com `PARAMETER num_thread 6` definido no Modelfile customizado
`llama3b-cpu` (ver [docs/04-modelos-download-execucao.md](04-modelos-download-execucao.md)),
o Ollama continuava rodando a inferência com as 8 threads disponíveis na VM, e não com as
6 esperadas.

**Diagnóstico:**
```bash
journalctl -u ollama | grep -i -- '--threads'
```
- O log confirmou que o processo era iniciado com todas as 8 threads, ignorando o valor
  `6` definido tanto no `PARAMETER num_thread` do Modelfile quanto no
  `OLLAMA_NUM_THREADS=6` já configurado no override do serviço (ver
  [docs/03-configuracao.md](03-configuracao.md)) — ou seja, nenhuma das duas formas de
  configuração em nível de aplicação estava sendo de fato respeitada nesta instalação.

**Solução: limitar por cgroup, no nível do sistema operacional.**

Como o controle "por dentro" do Ollama não se mostrou confiável, a solução foi restringir
a quais CPUs o processo do serviço tem acesso, deixando o sistema operacional impor o
limite em vez de depender do parâmetro da aplicação.

```bash
sudo systemctl edit ollama
```

Conteúdo adicionado ao override:
```ini
[Service]
AllowedCPUs=0-5
```

Aplicação da mudança:
```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

**Verificação:** com o `htop` aberto (ver [monitoramento.md](../../monitoramento.md), Zona 1),
rodando um novo teste, apenas 6 barras de CPU (0 a 5) mostraram atividade — confirmando
que o limite passou a ser respeitado.

**Por que funciona:** `AllowedCPUs` é uma diretiva do systemd que usa cgroups para
restringir, no nível do kernel, em quais CPUs as threads do serviço podem ser escalonadas
— diferente de `OLLAMA_NUM_THREADS`/`num_thread`, que dependem do Ollama respeitar o valor
internamente. Com `AllowedCPUs=0-5`, mesmo que o Ollama tente abrir mais threads do que
o configurado, o kernel simplesmente não permite que elas rodem fora das CPUs 0–5.

**Nota para a documentação:** isso significa que, neste ambiente, o controle de threads
efetivamente em vigor é o `AllowedCPUs=0-5` do override do systemd, e não o
`OLLAMA_NUM_THREADS`/`PARAMETER num_thread` documentados anteriormente — esses continuam
registrados (e não custa mantê-los, caso uma versão futura do Ollama passe a respeitá-los
corretamente), mas quem garante o limite hoje é o cgroup.
