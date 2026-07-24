# 2. Instalação do driver NVIDIA, CUDA e do Ollama

Registro dos comandos usados para instalar o driver NVIDIA, o CUDA Toolkit e o Ollama,
deixando o serviço rodando com a GPU Tesla V100S reconhecida.

## 2.1 Instalando o driver NVIDIA e o CUDA

### Adicione o repositório CUDA da NVIDIA

**Comandos:**
```bash
sudo dnf config-manager --add-repo \
  https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo

sudo dnf clean expire-cache
```

**Por quê:** esse repositório oficial da NVIDIA fornece tanto o driver quanto o CUDA Toolkit, com atualizações e a integração DKMS já empacotadas. Usar o repositório (em vez de um instalador `.run` avulso) facilita manutenção e updates.

### Instale o driver

Para a V100 (Volta, hardware de data center mais antigo), o **driver proprietário** é a escolha mais segura em termos de compatibilidade:

**Comandos:**
```bash
# Reseta qualquer stream de módulo pré-existente
sudo dnf module reset -y nvidia-driver

# Instala o driver proprietário mais recente + DKMS
sudo dnf module install -y nvidia-driver:latest-dkms
```

**Por quê o proprietário e não o `nvidia-open`:** os módulos de kernel "open" da NVIDIA são plenamente recomendados para arquiteturas mais novas. Na Volta, que é uma geração mais antiga, o driver proprietário tende a oferecer a compatibilidade mais previsível. Se você tiver um motivo específico para usar os módulos open, eles também funcionam na V100 — mas para "instalar e esquecer", o proprietário é o caminho de menor atrito.

### Instale o CUDA Toolkit

**Comandos:**
```bash
sudo dnf install -y cuda-toolkit-12-8
```

**Por quê:** o Ollama traz suas próprias bibliotecas de runtime CUDA compiladas, então tecnicamente ele roda sem o Toolkit completo. Mas ter o Toolkit instalado te dá o `nvidia-smi` sempre acessível, ferramentas de diagnóstico e compatibilidade para outras cargas. Vale o espaço em disco.

### Ajuste o PATH

O CUDA instala binários em `/usr/local/cuda/bin`, que não está no PATH padrão.

**Comandos:**
```bash
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

**Por quê:** sem isso, você pode instalar tudo corretamente e ainda ver `nvidia-smi: command not found`, o que gera pânico desnecessário. Ajustar o PATH resolve.

### Reinicie e verifique

**Comandos:**
```bash
sudo reboot
```

Após reiniciar:

```bash
nvidia-smi
```

**Saída / observações esperadas:** uma tabela listando `Tesla V100S`, a versão do driver, a versão do CUDA e ~32 GB (`32768 MiB`) de memória total. Se a placa aparece aqui, o alicerce está pronto.

**Problemas encontrados:** se a placa não aparecer, o problema está no driver, não no Ollama. Verifique se os kernel-headers batem com `uname -r`, cheque `dmesg | grep -i nvidia` e confirme que o DKMS compilou o módulo (`dkms status`).

## 2.2 Instalando o Ollama

### Instalação via script oficial

**Objetivo:** instalar o Ollama no Oracle Linux usando o script de instalação oficial, com detecção automática da GPU.

**Comandos:**
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

**Saída / observações:** o script detecta a distribuição, instala o binário, cria o usuário de sistema `ollama` e configura um serviço systemd automaticamente. Em ambiente de servidor, rodar como serviço (em vez de `ollama serve` num terminal) é o correto: sobe no boot, reinicia sozinho em caso de falha e roda de forma isolada. Com o driver NVIDIA já instalado (passo 2.1), o instalador detecta a Tesla V100S automaticamente.

**Problemas encontrados:** nenhum até o momento.

### Confirme a instalação

**Comandos:**
```bash
ollama --version
systemctl status ollama
```

**Saída / observações:** você deve ver o serviço `active (running)`. Nesse ponto o Ollama já enxerga sua GPU — mas ainda **sem** as otimizações específicas da V100S. Isso é tratado em [03-configuracao.md](03-configuracao.md).

**Problemas encontrados:** nenhum até o momento.
