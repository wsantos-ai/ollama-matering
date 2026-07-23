# 2. Instalação do Ollama

Registro dos comandos usados para instalar o Ollama e deixar o serviço rodando.

<!-- As etapas informadas pelo usuário serão adicionadas abaixo, nesta ordem cronológica -->

### Instalação do Ollama via script oficial

**Objetivo:** instalar o Ollama no Ubuntu usando o script de instalação oficial.

**Comandos:**
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

**Saída / observações:**
- O script detecta a distribuição/arquitetura, instala o binário `ollama` e configura o
  serviço systemd (`ollama.service`), já deixando-o habilitado e em execução.
- Em máquinas sem GPU, o próprio instalador identifica a ausência de GPU NVIDIA/AMD e
  configura o Ollama para rodar em modo CPU-only automaticamente.

**Problemas encontrados:** nenhum até o momento.

### Confirmação da instalação e do serviço (modo CPU)

**Objetivo:** verificar que o Ollama foi instalado corretamente e que o serviço está ativo,
rodando em modo CPU-only.

**Comandos:**
```bash
ollama --version
systemctl status ollama
```

**Saída / observações:**
- `ollama --version` confirma a versão do binário instalado.
- `systemctl status ollama` confirma que o serviço `ollama.service` está `active (running)`,
  habilitado no boot e gerenciado pelo systemd.

**Problemas encontrados:** nenhum até o momento.
