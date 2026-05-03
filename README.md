# poc-claro-hashicorp-win

POC de automação Ansible para gerenciamento de **Active Directory** no Windows, integrada ao **HashiCorp Vault** e ao **Ansible Automation Platform (AAP)**.

## Visão Geral

Este projeto automatiza operações no Active Directory usando a coleção `microsoft.ad`. As credenciais do Domain Controller são injetadas de forma segura pelo AAP por meio de uma credencial do tipo **Machine (Windows)** (`poc-claro-hashicorp-win-AD`), que recupera os valores diretamente do HashiCorp Vault antes da execução do playbook.

## O que o Playbook Faz

1. **Cria um grupo de segurança** no Active Directory (escopo Global, categoria Security).
2. **Cria um usuário** no Active Directory com UPN e OU configuráveis.
3. **Insere o usuário no grupo** criado anteriormente.
4. **Cria uma pasta** no servidor (`C:\Reports\POC_HashiCorp`).
5. **Gera um arquivo de relatório** (`poc_hashicorp_report.txt`) dentro da pasta com o log completo de todas as ações realizadas.
6. **Lê e exibe** o conteúdo do arquivo de relatório no output do Ansible.

## Estrutura do Projeto

```
poc-claro-hashicorp-win/
├── group_vars/
│   └── all.yaml                    # Variáveis globais (placeholder)
├── roles/
│   └── active_directory/
│       ├── tasks/
│       │   └── main.yaml           # Tarefas: grupo, usuário, pasta, relatório
│       └── vars/
│           └── main.yaml           # Variáveis: nomes, OUs, caminhos
├── ansible.cfg                     # Configuração do Ansible (callbacks, WinRM, timeouts)
├── active_directory.yaml           # Playbook principal
└── README.md
```

## Pré-requisitos

- Ansible >= 2.15
- Coleção `microsoft.ad` instalada:
  ```bash
  ansible-galaxy collection install microsoft.ad
  ```
- Coleção `ansible.windows` instalada:
  ```bash
  ansible-galaxy collection install ansible.windows
  ```
- WinRM habilitado no Domain Controller
- Credencial do tipo **Machine (Windows)** configurada no AAP, integrada ao HashiCorp Vault

## Credenciais e Integração com Vault

As credenciais **não são armazenadas** no repositório. O AAP injeta as seguintes variáveis de ambiente em tempo de execução, lidas a partir do HashiCorp Vault:

| Variável de Ambiente       | Descrição                                         |
|----------------------------|---------------------------------------------------|
| `AD_HOST`                  | Hostname/IP do Domain Controller                  |
| `AD_USER`                  | Usuário administrador do domínio                  |
| `AD_PASSWORD`              | Senha do administrador do domínio                 |
| `AD_USER_INITIAL_PASSWORD` | Senha inicial do usuário criado (opcional)        |

## Execução

### Via Ansible Automation Platform (AAP)

1. Importe o projeto no AAP apontando para este repositório.
2. Configure a credencial `poc-claro-hashicorp-win-AD` (tipo: Machine/Windows) com lookup no Vault.
3. Crie um **Job Template** utilizando o playbook `active_directory.yaml`.
4. Informe o inventário com o Domain Controller como host-alvo.
5. Execute o Job Template — as credenciais serão injetadas automaticamente.

### Via linha de comando (ambiente de teste com variáveis de ambiente)

> **Atenção:** Em produção, utilize sempre o AAP com credencial integrada ao Vault.
> As variáveis abaixo são usadas apenas como fallback no `AD_HOST`.

```bash
# O host-alvo é definido pelo inventário ou pela variável AD_HOST
# ansible_user e ansible_password devem ser passados via --extra-vars
# ou credencial de ambiente — nunca hardcoded no playbook.

ansible-playbook active_directory.yaml -i 10.10.10.3, \
  -e "ansible_user=lcsilva@lcs.local" \
  -e "ansible_password='SuaSenhaAqui'" \
  -e "ansible_winrm_transport=ntlm"
```

### Dry-run (check mode)

```bash
ansible-playbook active_directory.yaml --check
```

## Variáveis Configuráveis (`roles/active_directory/vars/main.yaml`)

| Variável               | Padrão                                    | Descrição                            |
|------------------------|-------------------------------------------|--------------------------------------|
| `ad_domain`            | `lcs.local`                                       | Nome do domínio AD                   |
| `ad_domain_dn`         | `DC=lcs,DC=local`                                 | Distinguished Name base do domínio   |
| `ad_group_name`        | `GRP_POC_HASHICORP`                               | Nome do grupo a ser criado           |
| `ad_group_scope`       | `Global`                                          | Escopo do grupo AD                   |
| `ad_group_category`    | `Security`                                        | Categoria do grupo AD                |
| `ad_group_ou`          | `OU=homelab,DC=lcs,DC=local`                      | OU onde o grupo será criado          |
| `ad_user_name`         | `usr_poc_hashicorp`                               | sAMAccountName do usuário            |
| `ad_user_display_name` | `POC HashiCorp User`                              | Nome de exibição do usuário          |
| `ad_user_ou`           | `OU=users,OU=homelab,DC=lcs,DC=local`             | OU onde o usuário será criado        |
| `ad_user_upn`          | `usr_poc_hashicorp@lcs.local`                     | User Principal Name                  |
| `ad_report_base_path`  | `C:\Reports\POC_HashiCorp`                        | Caminho da pasta de relatório        |
| `ad_report_file`       | `poc_hashicorp_report.txt`                        | Nome do arquivo de relatório         |

## Configurações do Ansible (`ansible.cfg`)

| Parâmetro               | Valor                                 | Descrição                                |
|-------------------------|---------------------------------------|------------------------------------------|
| `timeout`               | `120`                                 | Timeout de conexão em segundos           |
| `show_custom_stats`     | `true`                                | Exibe estatísticas customizadas ao final |
| `host_key_checking`     | `false`                               | Desabilita verificação de chave SSH      |
| `stdout_callback`       | `yaml`                                | Saída formatada em YAML                  |
| `callbacks_enabled`     | `timer, profile_tasks, profile_roles` | Métricas de tempo por task e role        |
| `winrm.transport`       | `ntlm`                                | Transporte WinRM padrão                  |