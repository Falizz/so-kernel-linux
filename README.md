# Laboratório de Sistemas Operacionais — IDP (2026/1)

## Trabalho Prático 01 (T01) — Customização de Kernel e Ambiente Embarcado em RAM

Este repositório contém a infraestrutura, os scripts de inicialização, os arquivos de configuração e o patch de código desenvolvidos para o Trabalho Prático 01 da disciplina de Sistemas Operacionais, ministrada pelo Professor Jeremias Moreira Gomes no IDP (Ciência da Computação / Engenharia de Software).

---

## 📋 Objetivo do Projeto

O objetivo deste projeto é construir, configurar e emular um Sistema Operacional Linux minimalista e independente, executado totalmente a partir da memória RAM (*initramfs*) através do emulador QEMU.

O fluxo principal do trabalho envolveu:

1. Compilação estática do **BusyBox** (v1.36.1) para prover os utilitários de espaço de usuário (*User Space*).
2. Configuração e compilação do **Kernel Linux** (v6.6.137).
3. Aplicação de um **patch customizado** (`kernel/sys.c`) contendo rotinas em Assembly inline atreladas à matrícula do aluno.
4. Disparo da chamada de sistema (`sysinfo`) de modo a descriptografar e expor o token de validação final via buffer do Kernel (`dmesg`).

---

## 🛠️ Arquitetura e Estrutura do Ambiente

O ambiente de desenvolvimento padrão utilizado foi baseado em:

| Item | Descrição |
|---|---|
| Hospedeiro | Windows 11 com WSL2 (Windows Subsystem for Linux) |
| Distribuição | Ubuntu 24.04 LTS |
| Arquitetura Alvo | x86-64 |

### Estrutura do Repositório

Conforme as boas práticas, os códigos-fontes massivos de terceiros (árvores completas do Linux e BusyBox) foram devidamente ignorados via `.gitignore`, mantendo o repositório leve, elegante e focado apenas no desenvolvimento autoral:

```
├── initramfs/
│   └── init                # Script customizado de inicialização do SO (Rootfs)
├── matricula.patch         # Patch Assembly injetado no arquivo kernel/sys.c
├── .gitignore              # Filtro para ignorar binários e as pastas gigantes de código
└── README.md                # Documentação do projeto
```

---

## 🚀 Passo a Passo de Execução e Configuração

### 1. Preparação das Dependências Básicas

Instalação dos compiladores, bibliotecas de manipulação e do emulador de hardware no Ubuntu:

```bash
sudo apt-get update
sudo apt-get install -y build-essential libncurses-dev bison flex libssl-dev libelf-dev pahole cpio
sudo apt-get install -y qemu-system qemu-utils qemu-system-x86
```

### 2. Espaço de Usuário Minimalista com BusyBox

O BusyBox unifica as ferramentas UNIX essenciais em um único binário.

```bash
git clone https://github.com/mirror/busybox.git
cd busybox
git checkout 1_36_1
make menuconfig
```

Configurações cruciais ajustadas no menu visual:

- **Settings → Build Options** → habilitar *Build static binary* (permite rodar sem dependências externas de bibliotecas dinâmicas).
- **Networking Utilities** → desabilitar a ferramenta `tc` (legada, incompatível com a versão atual do Kernel).

```bash
make -j2
make install
```

Isso gerou a pasta local `_install/` contendo a base do espaço de usuário.

### 3. Configuração, Patching e Compilação do Kernel

```bash
cd ~/idp
git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git
cd linux-stable
git checkout v6.6.137
make menuconfig
```

Configurações exigidas na interface do Kernel:

- **General setup** → ativar *Initial RAM filesystem and RAM disk (initramfs/initrd) support*.
- **64-bit kernel** → habilitado por padrão.
- **Device Drivers → Generic Driver Options** → ativar *Maintain a devtmpfs filesystem to mount at /dev* e *Automount devtmpfs at /dev*.

**Ajuste de assinatura de drivers**, para evitar que a compilação local quebre exigindo certificados de chaves de segurança ausentes:

```bash
scripts/config --disable SYSTEM_TRUSTED_KEYS
scripts/config --disable SYSTEM_REVOCATION_KEYS
```

> **Nota:** caso ocorra erro de *target 'y'* na pasta de certificados devido à formatação de strings vazias, a correção é aplicada de forma cirúrgica limpando as entradas do arquivo `.config`.

**Aplicação do patch e compilação:**

O arquivo `matricula.patch` modificou diretamente a função `do_sysinfo` em `kernel/sys.c` para ler a versão do sistema e realizar cálculos computacionais em registradores da CPU:

```bash
git apply ../matricula.patch
make -j2 bzImage
```

### 4. Montagem do Sistema de Arquivos em RAM (initramfs)

Criação manual da árvore de diretórios padrão que o Linux espera ao inicializar, acompanhada do script `init` que dita as instruções primárias de montagem dos sistemas de arquivos virtuais (`/proc`, `/sys`, `/dev`):

```bash
cd ~/idp
mkdir initramfs && cd initramfs
mkdir -p bin sbin etc proc sys dev usr/bin usr/sbin
cp -a ~/idp/busybox/_install/* .
```

O arquivo `init` foi gerado na raiz da pasta e recebeu permissão de execução (`chmod +x init`), contendo as chamadas primárias do sistema e exibição do tempo de boot:

```sh
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mknod -m 666 /dev/ttyS0 c 4 64
echo -e "\nBem-vindo ao IDP Linux!\n\n"
echo -e "Boot levou $(cut -d' ' -f1 /proc/uptime) segundos!\n\n"
setsid cttyhack sh
poweroff -f
exec /bin/sh
```

Geração dos nós de hardware fictícios cruciais para a comunicação básica e empacotamento em formato comprimido compatível com o Kernel:

```bash
sudo mknod -m 622 dev/console c 5 1
sudo mknod -m 666 dev/null c 1 3
sudo mknod -m 666 dev/zero c 1 5
sudo mknod -m 666 dev/ptmx c 5 2
sudo mknod -m 666 dev/tty c 5 0
sudo mknod -m 444 dev/random c 1 8
sudo mknod -m 444 dev/urandom c 1 9
sudo chown root:tty dev/{console,ptmx,tty}

find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz
```

---

## 🎯 Emulação no QEMU e Coleta do Token Final

Com a imagem de boot (`bzImage`) gerada e o disco em RAM pronto, o sistema foi carregado de forma limpa, isolada e sem interface gráfica direto no terminal:

```bash
cd ~/idp
qemu-system-x86_64 -kernel linux-stable/arch/x86_64/boot/bzImage -initrd initramfs.cpio.gz -nographic -append "console=ttyS0 loglevel=3"
```

### Validação do Boot

O sistema operacional inicializou com sucesso total em 2.51 segundos:

```
Bem-vindo ao IDP Linux!
Boot levou 2.51 segundos!
~ #
```

### Disparo da Syscall e Resolução

Como o ambiente não dispõe de bibliotecas C dinâmicas padrão para compilação instantânea, utilizou-se a estratégia de comandos nativos do sistema (`free` ou `uptime`). Ambos os utilitários invocam obrigatoriamente a chamada de sistema `sysinfo` por baixo dos panos para capturar os dados do sistema, forçando o Kernel a passar pelo bloco do patch modificado.

```
~ # free
~ # dmesg | grep SOLUCAO
[   39.561840] [+] SOLUCAO: C1B9344556677C4
```

---

## 🏆 Token Final Gerado (Moodle)

O código hexadecimal único gerado com sucesso por meio do processamento do bloco Assembly inline no coração do Kernel foi:

```
C1B9344556677C4
```

*Desenvolvido como parte dos requisitos práticos da disciplina de Sistemas Operacionais (IDP, 2026).*
