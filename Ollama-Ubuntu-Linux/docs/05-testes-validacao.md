# 5. Testes e validação

Smoke tests via CLI e via API HTTP do Ollama, verificação de que os modelos respondem
corretamente.

<!-- As etapas informadas pelo usuário serão adicionadas abaixo, nesta ordem cronológica -->

### Verificação de modelos carregados (`ollama ps`)

**Objetivo:** verificar quais modelos estão atualmente carregados em memória, com qual
processador (CPU/GPU) estão rodando e por quanto tempo ainda ficarão carregados.

**Comandos:**
```bash
ollama ps
```

**Saída / observações:**
```
NAME          ID             SIZE     PROCESSOR   CONTEXT   UNTIL
llama3.2:3b   a80c4f17acd5   2.5 GB   100% CPU    4096      23 minutes from now
```
- `PROCESSOR: 100% CPU` confirma que o modelo está rodando inteiramente em CPU, sem
  nenhum offload de camadas para GPU — coerente com o ambiente sem GPU deste projeto.
- `CONTEXT: 4096` é a janela de contexto padrão do `llama3.2:3b` original (diferente do
  modelo customizado `llama3b-cpu`, criado com `num_ctx 8192` — ver
  [docs/04-modelos-download-execucao.md](04-modelos-download-execucao.md)). Ou seja, essa
  execução específica foi feita com o modelo base, não com a variante customizada.
- `UNTIL: 23 minutes from now` reflete o `OLLAMA_KEEP_ALIVE=60m` configurado no serviço
  (ver [docs/03-configuracao.md](03-configuracao.md)): o modelo permanece carregado em
  RAM por até 60 minutos após o último uso, e essa contagem regressiva mostra quanto
  tempo falta até ele ser descarregado da memória caso não haja nova requisição.

**Problemas encontrados:** nenhum até o momento.

### Acompanhamento de logs do serviço (`journalctl`)

**Objetivo:** acompanhar os logs do Ollama para validar o comportamento do serviço (ex.:
confirmar a correção do limite de threads via `AllowedCPUs`, ver
[docs/07-troubleshooting.md](07-troubleshooting.md)) tanto em tempo real quanto olhando
para trás.

**Comandos:**
```bash
journalctl -u ollama -f          # acompanhar em tempo real
journalctl -u ollama --since "1 hour ago"
```

**Saída / observações:**
- `-f` ("follow") mantém o terminal aberto exibindo novas linhas de log conforme são
  geradas — útil para observar o serviço enquanto se roda um teste em outra janela (ex.:
  o script de benchmark de threads).
- `--since "1 hour ago"` filtra o histórico de log para a última hora, útil para revisar o
  que aconteceu (reinícios do serviço, erros, mudanças de configuração aplicadas) sem
  precisar rolar o log inteiro desde o boot da máquina.
- Esse comando foi especialmente útil com `OLLAMA_DEBUG=1` ligado (ver
  [docs/03-configuracao.md](03-configuracao.md)), quando o volume de log era maior e havia
  mais detalhe para conferir.

**Problemas encontrados:** nenhum até o momento.
