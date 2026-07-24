# 3. Configuração

Variáveis de ambiente, override do serviço systemd e ajustes de firewall para otimizar o
Ollama na Tesla V100S.

## 3.1 Configurando o serviço com as variáveis de otimização

Aqui está o ponto onde a maioria das pessoas erra. **As variáveis de ambiente do Ollama são lidas pelo servidor (`ollama serve`) na inicialização — não pelo comando `ollama run`.** Definir a variável no seu terminal não tem efeito nenhum se o servidor já está rodando como serviço systemd, porque o serviço tem o próprio ambiente. Portanto, configuramos **no systemd**.

### Crie um override do serviço

**Objetivo:** aplicar variáveis de ambiente de otimização de GPU/memória sem editar o unit file original.

**Comandos:**
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

**Problemas encontrados:** nenhum até o momento.

### Entenda cada variável (o "porquê" de cada linha)

**`OLLAMA_HOST=0.0.0.0:11434`**
Faz o Ollama escutar em todas as interfaces de rede, permitindo que seu orquestrador de agentes acesse a API de outra máquina. ⚠️ **Atenção de segurança:** isso expõe a API sem autenticação. Só use em rede confiável e **sempre** com firewall restringindo a porta 11434 à sua sub-rede (veja 3.2). Se o orquestrador roda na mesma máquina do Ollama, deixe `127.0.0.1:11434` (o padrão) e pule o firewall.

**`OLLAMA_FLASH_ATTENTION=1`**
Ativa o suporte a Flash Attention. Como visto em [01-prerequisitos-e-ambiente.md](01-prerequisitos-e-ambiente.md), na V100 o efeito é por-modelo, mas ligar não custa nada e ajuda nos modelos compatíveis, reduzindo VRAM da atenção e acelerando contextos longos — exatamente o cenário agêntico.

**`OLLAMA_KV_CACHE_TYPE=f16`**
Define o tipo de dado do KV cache. Usamos `f16` (float16) porque a V100 **não** tem bfloat16 nativo, e f16 é o formato bem suportado pela Volta. Existem opções mais agressivas (`q8_0`, `q4_0`) que quantizam o próprio cache para economizar ainda mais VRAM — porém a quantização de KV cache no llama.cpp geralmente **depende de Flash Attention ativo**, o que na V100 é inconsistente. Por isso começamos com `f16`, que é o mais seguro; você pode experimentar `q8_0` depois se precisar de mais folga e confirmar que a qualidade se mantém.

**`OLLAMA_MAX_LOADED_MODELS=2`**
Permite manter até 2 modelos carregados em VRAM ao mesmo tempo. Esse é o coração da estratégia agêntica: um modelo pequeno "roteador" fica sempre residente enquanto o modelo pesado da tarefa entra e sai. Com 32 GB você tem espaço para isso.

**`OLLAMA_NUM_PARALLEL=1`**
Quantas requisições simultâneas cada modelo processa. Cada slot paralelo aloca seu próprio KV cache, consumindo VRAM adicional. Numa única GPU, manter em `1` (ou no máximo `2`) deixa o consumo de memória previsível e evita OOM. Deixe seu orquestrador serializar as chamadas pesadas.

**`OLLAMA_KEEP_ALIVE=10m`**
Por quanto tempo um modelo fica na VRAM após a última requisição. O padrão é 5 minutos. Em uso agêntico, subir para 10m evita recarregar do disco a cada troca de tarefa (o carregamento custa segundos e é sentido na latência). Em [04-modelos-download-execucao.md](04-modelos-download-execucao.md) refinamos isso por-chamada.

**`OLLAMA_DEBUG=1`**
Faz o Ollama logar cada variável resolvida e detalhes de carregamento de cada modelo — inclusive se o Flash Attention ficou `Enabled` ou `Disabled`. **Essencial agora**, no começo, para você validar que tudo pegou. Depois de confirmar (veja [05-testes-validacao.md](05-testes-validacao.md)), remova para deixar os logs limpos.

### Aplique as mudanças

**Comandos:**
```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

**Saída / observações:** o `daemon-reload` faz o systemd reler a configuração do serviço; o `restart` reinicia o Ollama para que ele leia as novas variáveis (lembre: variáveis só são lidas na inicialização).

**Problemas encontrados:** nenhum até o momento.

## 3.2 (Se usar acesso remoto) Configure o firewall

**Objetivo:** restringir o acesso à API do Ollama (porta 11434) apenas à sub-rede confiável, caso `OLLAMA_HOST` esteja exposto em `0.0.0.0`.

**Comandos:**
```bash
# Libera a porta apenas para a sua sub-rede (ajuste o CIDR)
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="11434" protocol="tcp" accept'
sudo firewall-cmd --reload
```

**Saída / observações:** sem isso, expor `0.0.0.0` deixa qualquer um na rede usar (e abusar) da sua GPU. A regra restringe o acesso à sua sub-rede confiável.

**Problemas encontrados:** nenhum até o momento.
