# 5. Testes e validação

Smoke tests, verificação de residência em VRAM, monitoramento em tempo real e validação de
que o Flash Attention está de fato ativo por modelo.

Otimização sem medição é só suposição. Aqui está como confirmar que está tudo funcionando como planejado.

## 5.1 Confirme o que está carregado na VRAM

**Objetivo:** verificar quais modelos estão em memória, quanto ocupam e se estão 100% na GPU.

**Comandos:**
```bash
ollama ps
```

**Saída / observações:** mostra quais modelos estão em memória, quanto ocupam, se estão 100% na GPU ou divididos com CPU, e quando serão descarregados. **Se aparecer qualquer coisa diferente de 100% GPU, você tem offload para CPU** — hora de reduzir contexto ou usar quantização mais leve.

**Problemas encontrados:** nenhum até o momento.

## 5.2 Monitore a GPU em tempo real

**Comandos:**
```bash
watch -n 1 nvidia-smi
```

**Saída / observações:** acompanhe o uso de VRAM e a utilização do núcleo (`GPU-Util`). Durante a geração, o `GPU-Util` deve subir bem. Se a VRAM estiver perto do limite dos 32 GB, você está no fio da navalha para OOM — deixe uma margem.

## 5.3 Onde consultar os logs

Como o Ollama roda via systemd no Oracle Linux:

**Comandos:**
```bash
# Todos os logs do serviço
journalctl -u ollama

# Acompanhar em tempo real (ótimo durante testes)
journalctl -u ollama -f

# Só a última hora
journalctl -u ollama --since "1 hour ago"
```

## 5.4 Valide o Flash Attention (o teste da V100)

**Objetivo:** confirmar, por modelo, se o Flash Attention realmente ficou ativo — na V100 isso não é garantido mesmo com a variável ligada (veja [01-prerequisitos-e-ambiente.md](01-prerequisitos-e-ambiente.md)).

Com `OLLAMA_DEBUG=1` ativo, carregue um modelo e procure nos logs a linha de carregamento:

**Comandos:**
```bash
journalctl -u ollama -f | grep -i flash
```

**Saída / observações:** você verá algo como `FlashAttention:Enabled` ou `FlashAttention:Disabled` no registro de load daquele modelo. **Faça esse teste para cada modelo do seu elenco.** Se ficar `Disabled` num modelo importante, você saberá que não pode contar com a economia de VRAM da atenção para ele — e deve dimensionar o contexto de forma mais conservadora.

**Problemas encontrados:** (registrar aqui quais modelos ficaram `Enabled` vs `Disabled` conforme forem testados)

## 5.5 Desligue o debug depois

**Objetivo:** reduzir o volume de logs após a validação inicial.

Uma vez validado tudo, edite o serviço (`sudo systemctl edit ollama`), remova a linha `OLLAMA_DEBUG=1`, e reinicie:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

**Por quê:** logs de debug são verbosos demais para operação contínua.
