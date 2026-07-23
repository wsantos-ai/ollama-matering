# 1. Pré-requisitos e ambiente

Registro do hardware, sistema operacional e checagens feitas antes de instalar o Ollama.

<!-- As etapas informadas pelo usuário serão adicionadas abaixo, nesta ordem cronológica -->

### Atualização de pacotes e instalação do curl

**Objetivo:** deixar o sistema com os pacotes atualizados e garantir a presença do `curl`,
necessário para baixar o script de instalação do Ollama na próxima etapa.

**Comandos:**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl
```

**Saída / observações:**
- `apt update` atualiza os índices de pacotes disponíveis; `apt upgrade -y` aplica as
  atualizações sem pedir confirmação interativa.
- `curl` é pré-requisito comum em instalações de Ubuntu Server minimalistas — em algumas
  imagens já vem instalado, mas o comando é idempotente (não há problema em rodar mesmo se
  já existir).

**Problemas encontrados:** nenhum até o momento.
