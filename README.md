# Laboratório de Sistemas Operacionais — IDP (2026/1)
## Trabalho Prático 01 (T01) — Customização de Kernel e Ambiente Embarcado em RAM

Este repositório contém a infraestrutura, os scripts de inicialização, os arquivos de configuração e o patch de código desenvolvidos para o **Trabalho Prático 01** da disciplina de **Sistemas Operacionais**, ministrada pelo **Professor Jeremias Moreira Gomes** no IDP (Ciência da Computação / Engenharia de Software).

---

## 📋 Objetivo do Projeto

O objetivo deste projeto é construir, configurar e emular um **Sistema Operacional Linux minimalista e independente**, executado totalmente a partir da memória RAM (**initramfs**) através do emulador **QEMU**. 

O fluxo principal do trabalho envolveu:
1. Compilação estática do **BusyBox (v1.36.1)** para prover os utilitários de espaço de usuário (User Space).
2. Configuração e compilação do **Kernel Linux (v6.6.137)**.
3. Aplicação de um **patch customizado** (`kernel/sys.c`) contendo rotinas em **Assembly inline** atreladas à matrícula do aluno (`2321055`).
4. Disparo da chamada de sistema (`sysinfo`) de modo a descriptografar e expor o token de validação final via buffer do Kernel (`dmesg`).

---

## 🛠️ Arquitetura e Estrutura do Ambiente

O ambiente de desenvolvimento padrão utilizado foi baseado em:
* **Hospedeiro:** Windows 11 com WSL2 (Windows Subsystem for Linux).
* **Distribuição:** Ubuntu 24.04 LTS.
* **Arquitetura Alvo:** x86-64.

### Estrutura do Repositório (Foco no Trabalho)
Conforme as boas práticas de desenvolvimento, os códigos-fontes massivos de terceiros (árvores completas do Linux e BusyBox) foram devidamente ignorados via `.gitignore`, mantendo o repositório leve, elegante e focado apenas no desenvolvimento autoral:

```text
├── initramfs/
│   └── init                # Script customizado de inicialização do SO (Rootfs)
├── matricula.patch         # Patch Assembly injetado no arquivo kernel/sys.c
├── .gitignore              # Filtro para ignorar binários e as pastas gigantes de código
└── README.md               # Documentação do projeto