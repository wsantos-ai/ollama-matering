# 7. Troubleshooting

Erros específicos da Tesla V100S/Oracle Linux encontrados durante o processo e como
resolvê-los.

<!-- Novos problemas encontrados serão adicionados abaixo, nesta ordem cronológica -->

## 7.0 `No match for argument: kernel-devel-...el9uek.x86_64`

**Sintoma:** ao rodar `sudo dnf install -y kernel-devel-$(uname -r) kernel-headers-$(uname -r) gcc make` (ver [01-prerequisitos-e-ambiente.md](01-prerequisitos-e-ambiente.md)):
```
No match for argument: kernel-devel-6.12.0-204.92.4.3.1.el9uek.x86_64
No match for argument: kernel-headers-6.12.0-204.92.4.3.1.el9uek.x86_64
Error: Unable to find a match: kernel-devel-... kernel-headers-...
```

**Causa:** o kernel ativo é o **UEK** (Unbreakable Enterprise Kernel, padrão do Oracle Linux — identificável pelo sufixo `.el9uek` em `uname -r`). Os pacotes `kernel-devel`/`kernel-headers` (sem `uek`) pertencem ao kernel RHCK e não existem com esse sufixo de versão.

**Solução:** instalar o pacote equivalente para UEK:
```bash
sudo dnf install -y kernel-uek-devel-$(uname -r) gcc make
```
`gcc` e `make` costumam já estar instalados nesse ponto (confirmado pela saída `Package ... already installed`), então o comando efetivamente necessário é só o `kernel-uek-devel`. Detalhes completos em [01-prerequisitos-e-ambiente.md](01-prerequisitos-e-ambiente.md#instale-os-headers-do-kernel-em-uso).

## 7.1 Erro `CUDA error: device kernel image is invalid`

**Causa:** incompatibilidade de certas versões do Ollama/llama.cpp com a arquitetura Volta.

**Solução:**
1. Atualize para a versão mais recente do Ollama (correções para GPUs antigas têm saído continuamente):
   ```bash
   curl -fsSL https://ollama.com/install.sh | sh
   sudo systemctl restart ollama
   ```
2. Se persistir, teste temporariamente uma versão anterior conhecida por funcionar com V100 e acompanhe as issues do repositório do Ollama no GitHub até a correção estabilizar.

## 7.2 Modelo caiu para CPU (lento)

**Sintoma:** `ollama ps` mostra split CPU/GPU; geração lenta.

**Causas e soluções em ordem:**
1. Contexto grande demais → reduza `num_ctx` no Modelfile (ver [04-modelos-download-execucao.md](04-modelos-download-execucao.md)).
2. Quantização pesada demais → troque `q8_0`/`q5_K_M` por `q4_K_M`.
3. Outro modelo ocupando VRAM → confira `ollama ps` e ajuste `OLLAMA_MAX_LOADED_MODELS` ou o `keep_alive` (ver [03-configuracao.md](03-configuracao.md)).

## 7.3 Erro mencionando `bfloat16`

**Causa:** algo tentou usar bf16, que a V100 não suporta nativamente (ver [01-prerequisitos-e-ambiente.md](01-prerequisitos-e-ambiente.md)).

**Solução:** com Ollama isso é raro (o GGUF cuida disso), mas se ocorrer com outra ferramenta, force o uso de `float16`/`half`. No Ollama, garanta `OLLAMA_KV_CACHE_TYPE=f16`.

## 7.4 GPU some após suspend/resume (Linux)

**Sintoma:** após o sistema hibernar e voltar, o Ollama não enxerga mais a GPU.

**Solução:** recarregue o módulo UVM da NVIDIA:
```bash
sudo rmmod nvidia_uvm && sudo modprobe nvidia_uvm
```
Em servidor de produção, o ideal é desabilitar suspensão automática para evitar isso.

## 7.5 `nvidia-smi: command not found`

**Causa:** PATH não inclui os binários do CUDA.

**Solução:** revise o passo "Ajuste o PATH" em [02-instalacao-ollama.md](02-instalacao-ollama.md).

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

### Princípios gerais

- **VRAM é o recurso mais escasso.** Toda decisão (quantização, contexto, offload, quantos modelos residentes) é uma disputa por VRAM.
- **100% na GPU é a meta.** Offload para CPU é 3–5x mais lento; prefira quantização mais leve a aceitar offload.
- **Meça, não suponha.** `ollama ps`, `nvidia-smi` e os logs contam a verdade.
- **Conheça seu hardware.** Na V100, bfloat16 não existe e Flash Attention é por-modelo — projete com essas restrições em mente.
- **No fluxo agêntico, o custo é a troca de modelo.** Roteador residente + especialistas sob demanda, com `keep_alive` controlado por chamada, é o padrão que mantém a latência baixa numa única GPU.
