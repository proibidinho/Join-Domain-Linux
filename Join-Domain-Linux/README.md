# Join-Domain-Linux

Playbook Ansible para **inserir VMs Linux no domínio Active Directory `corp.clarobr`**, com suporte a:

- **RHEL 8 e 9** (registro no Satellite)
- **Oracle Linux 8 e 9** (sem Satellite)

> RHEL 7 / Oracle Linux 7 **não são mais suportados** pelo time.

---

## Sumário rápido

| O que | Onde |
|---|---|
| Ponto de entrada | `playbook.yaml` |
| Configuração do AD/Satellite | `roles/<role>/defaults/main.yml` |
| Senha do usuário de teste | `roles/<role>/defaults/main.yml` (`test_password`) |
| Grupos permitidos no SSH | `roles/<role>/defaults/main.yml` (`sshd_allowgroups`) — **AllowGroups está comentado** no template até o time de Linux definir |
| Templates corporativos | `roles/<role>/templates/` |
| Credenciais do CyberArk | obtidas via `lookup('cyberark_claro')` em runtime |
| Katello CA (Satellite) | precisa estar em `/tmp/katello-ca-consumer-latest.noarch.rpm` no Execution Node |

---

## Estrutura

```
.
├── ansible.cfg
├── playbook.yaml
├── README.md
└── roles
    ├── inventory/            # adiciona VMs ao grupo dinâmico 'unix'
    ├── detect-os/            # lê /etc/os-release via 'raw' e agrupa por SO
    ├── bootstrap-python/     # garante Python 3 instalado (pre_task)
    ├── rhel8plus/            # RHEL 8/9 - inclui registro no Satellite
    │   ├── defaults/
    │   ├── handlers/
    │   ├── tasks/
    │   └── templates/
    └── ol8plus/              # Oracle Linux 8/9 - sem RHSM
        ├── defaults/
        ├── handlers/
        ├── tasks/
        └── templates/
```

---

## Fluxo de execução

```
1. Etapa 1 (localhost) ─ role inventory
       └─ recebe a variável 'servers' (-e "servers=...") e adiciona ao grupo 'unix'

2. Etapa 2 (hosts: unix) ─ role detect-os
       └─ via 'raw' lê /etc/os-release e cria os grupos:
            - linux_rhel8plus   (ID=rhel,   VERSION>=8)
            - linux_ol8plus     (ID=ol/oraclelinux, VERSION>=8)
            - linux_unsupported (qualquer outro caso)

3. Etapa 3a (hosts: linux_rhel8plus)
       ├─ pre_task: role bootstrap-python   ← garante python3 instalado
       └─ role rhel8plus                    ← Satellite + AD

4. Etapa 3b (hosts: linux_ol8plus)
       ├─ pre_task: role bootstrap-python   ← garante python3 instalado
       └─ role ol8plus                      ← AD (sem Satellite)

5. Etapa 4 (hosts: linux_unsupported)
       └─ falha explícita com mensagem clara
```

---

## Role `bootstrap-python`

Garante que algum Python 3 esteja instalado **antes** de qualquer módulo nativo do Ansible rodar. Necessário porque servidores RHEL/OL 8/9 minimal podem vir sem `python3`.

Ordem de preferência:

1. `python3.9` (pacote `python39` ou `python3.9`)
2. `python3.8` (pacote `python38` ou `python3.8`)
3. `python3` (genérico)

Tudo executado via `raw` (não exige Python no alvo). Se nenhuma das tentativas funcionar, o playbook **falha de forma clara**.

---

## Role `rhel8plus`

Executa em ordem:

1. Coleta facts mínimos (`distribution`, `date_time`)
2. **Satellite/RHSM**: limpa registro anterior → instala `katello-ca-consumer` (transferido via base64 do Execution Node) → faz `subscription-manager register` com a activation key correta da versão (`AK-RHEL8` ou `AK-RHEL9`)
3. Desabilita SELinux (runtime + persistente) e `firewalld`
4. Instala pacotes do AD via `dnf` CLI (não exige `python3-dnf`):
   - `realmd sssd sssd-tools oddjob oddjob-mkhomedir adcli samba-common samba-common-tools krb5-workstation authselect`
5. Busca credenciais no **CyberArk** (`lookup('cyberark_claro')`)
6. Verifica se o host já está no domínio. Se não, remove keytab antigo e faz `adcli join` com password via stdin (com 3 retries)
7. Valida que `/etc/krb5.keytab` foi criado
8. Renderiza `/etc/krb5.conf` e `/etc/sssd/sssd.conf`
9. Cria diretório `/etc/ssh/sshd_config.d/` e aplica drop-in `aap.conf` (validado por `sshd -t -f`)
10. Garante `Include /etc/ssh/sshd_config.d/*.conf` no `sshd_config` principal
11. Aplica drop-in `/etc/sudoers.d/aap` (validado por `visudo -c -f`)
12. Para o SSSD, **limpa cache** (`/var/lib/sss/db/*` e `/var/lib/sss/mc/*`) — evita erro `could not initialize backend`
13. `authselect check` → `authselect apply-changes` (se inconsistente) → `authselect select sssd with-mkhomedir --force`
14. **Enable + start** dos serviços (`sssd`, `oddjobd`, `sshd`, `realmd`)
15. **Restart sequencial** dos serviços na ordem `realmd → oddjobd → sshd → sssd`
    > Esta ordem ajuda o `realm list` a popular corretamente em alguns hosts
16. **Pause de 10s** para estabilizar
17. Validações finais:
    - `realm list` deve conter `corp.clarobr` (com retries)
    - `getent passwd USER@DOMAIN` (com retries)
    - `id USER@DOMAIN`
    - `kinit USER@DOMAIN.UPPER` com `KRB5CCNAME` isolado

## Role `ol8plus`

**Idêntico ao `rhel8plus` exceto que NÃO faz Satellite/RHSM.**

---

## Variáveis configuráveis

### `roles/rhel8plus/defaults/main.yml`

```yaml
ad_domain: corp.clarobr
ad_dc: caadscp05.corp.clarobr

# Usuário de teste pós-join
test_user: z601427
test_password: "Dev!lmc12"

# Grupos permitidos no AllowGroups (atualmente COMENTADO no template).
# Descomente a linha AllowGroups no template quando o time de Linux definir.
sshd_allowgroups: "wheel root PS_crk_aap_admin PScrkaapadmin"

# Registro no Satellite/RHSM
rhsm_org: "CLAROBR"
rhsm_activation_key_rhel8: "AK-RHEL8"
rhsm_activation_key_rhel9: "AK-RHEL9"

# Caminho do pacote katello-ca-consumer no Execution Node (controller)
katello_ca_rpm_local: /tmp/katello-ca-consumer-latest.noarch.rpm
```

### `roles/ol8plus/defaults/main.yml`

```yaml
ad_domain: corp.clarobr
ad_dc: caadscp05.corp.clarobr
test_user: z601427
test_password: "Dev!lmc12"
sshd_allowgroups: "wheel root PS_crk_aap_admin PScrkaapadmin"
```

Todas podem ser sobrescritas via `-e var=valor` na linha de comando.

---

## Como executar

```bash
# Passar lista de servidores como string (1 por linha)
ansible-playbook playbook.yaml \
  -e "servers=$(cat lista_de_hosts.txt)"

# Sobrescrever variáveis sem editar o defaults
ansible-playbook playbook.yaml \
  -e "servers=host1
host2" \
  -e 'sshd_allowgroups=wheel root PS_crk_aap_admin PScrkaapadmin MEU_GRUPO_TESTE' \
  -e 'rhsm_activation_key_rhel8=AK-RHEL8-DEV'
```

---

## Pré-requisitos no Execution Node (controller)

- Lookup plugin `cyberark_claro` instalado e funcional
- Pacote `katello-ca-consumer-latest.noarch.rpm` presente em `/tmp/` (apenas para hosts RHEL)
- Conectividade SSH aos hosts alvo com usuário sudo/root
- Resolução DNS do domínio `corp.clarobr` e dos DCs `CADNS01/02.corp.clarobr`

---

## Tratativas de erro embutidas

| Cenário | Como é tratado |
|---|---|
| Host sem Python | Role `bootstrap-python` instala `python3.9 → 3.8 → python3` via `raw` |
| `python3-dnf` ausente | Não usamos módulo `dnf`; usamos `command: dnf …` direto |
| `realm list` vazio após join | **Restart sequencial** `realmd → oddjobd → sshd → sssd` + `pause` de 10s |
| Erro `could not initialize backend` no kinit | Limpa cache SSSD (`/var/lib/sss/db/*` e `/mc/*`) antes do restart; `kinit` usa `KRB5CCNAME` isolado |
| Erro `failed to read keytab` | `/etc/krb5.keytab` é removido antes do (re)join; validado com `stat` depois |
| Certificado do Satellite quebrado | `rpm -Uvh --force --nodigest --nosignature` no katello-ca + `failed_when: false` |
| `adcli join` falhando | 3 retries com 15s de delay |
| `subscription-manager register` falhando | 3 retries com 10s de delay |
| Host já está no domínio | `realm list` é verificado antes; join só ocorre se necessário (idempotente) |
| Erro de Python 3.8 "no start of json char found" | `interpreter_python=auto_silent` no `ansible.cfg` |

---

## Manutenção

### Adicionar novo grupo ao AllowGroups (SSH)

1. Edite `roles/rhel8plus/defaults/main.yml` (e `ol8plus/`) na linha `sshd_allowgroups`
2. Edite `roles/rhel8plus/templates/sshd_config.j2` (e `ol8plus/`):
   - **Descomente** a linha `# AllowGroups {{ sshd_allowgroups }}`

### Trocar a activation key do Satellite

Edite `roles/rhel8plus/defaults/main.yml`:
- `rhsm_activation_key_rhel8` → chave da RHEL 8
- `rhsm_activation_key_rhel9` → chave da RHEL 9

Ou passe via extra-var: `-e rhsm_activation_key_rhel8=NOVA_CHAVE`.

### Trocar o domínio/DC do AD

Edite `roles/<role>/defaults/main.yml`:
- `ad_domain`
- `ad_dc`

Também é necessário ajustar `roles/<role>/templates/krb5.conf.j2` se os nomes dos KDCs forem diferentes do padrão `CADNS01/02.<ad_domain>`.

### Adicionar suporte a outro SO

1. Adicione o ID/versão em `roles/detect-os/tasks/main.yaml` no `group_by`
2. Crie `roles/<novo>/` com mesma estrutura (`defaults/`, `handlers/`, `tasks/`, `templates/`)
3. Adicione a etapa correspondente em `playbook.yaml`

---

## Troubleshooting rápido

| Sintoma | Diagnóstico / Ação |
|---|---|
| `realm list` retorna vazio após o playbook | Já tratado: o restart sequencial e o pause cobrem isso. Se persistir, rodar manualmente `systemctl restart realmd oddjobd sshd sssd && sleep 10 && realm list` |
| `kinit: Cannot find KDC for realm "CORP.CLAROBR"` | Verificar `/etc/krb5.conf` (deve ter `kdc = CADNS01.corp.clarobr` etc) e DNS |
| `kinit: Preauthentication failed` | Senha em `test_password` está incorreta — verificar com o time de AD |
| `Failed to read keytab` | Rodar o playbook novamente: ele remove keytab antigo e refaz join |
| Satellite recusando registro com erro de cert | Atualizar `/tmp/katello-ca-consumer-latest.noarch.rpm` no Execution Node com a versão mais recente do Satellite |
| `module result deserialization failed - no start of JSON char found` | Já tratado em `ansible.cfg` (`interpreter_python=auto_silent`). Se persistir em algum host, passar `-e ansible_python_interpreter=/usr/libexec/platform-python` |
| Playbook falha em "bootstrap python" | Verificar se o host tem repositório dnf/yum funcional. Em RHEL precisa estar registrado no Satellite ANTES (mas o playbook já tenta antes — pode ser inacessibilidade de rede) |

---

## Idempotência

O playbook é **seguro para rodar múltiplas vezes** no mesmo host:

- `realm list` é checado antes do join → não rejoinha se já estiver no domínio
- `subscription-manager` faz `--force` no register (sempre re-registra, idempotente)
- `template` com `validate` só substitui se o conteúdo for válido
- `lineinfile` só adiciona se a regex não bater
- Drop-ins em `/etc/sudoers.d/` e `/etc/ssh/sshd_config.d/` são sempre regenerados

---

## Histórico de decisões importantes

| Decisão | Motivo |
|---|---|
| Roles separados `rhel8plus` e `ol8plus` | Oracle Linux **não** usa Satellite/RHSM. Mantém o código de cada plataforma isolado e simples. |
| `bootstrap-python` como role separado | Facilita reuso e leitura. Roda como `pre_task` antes de cada role principal. |
| `command: dnf …` em vez do módulo `ansible.builtin.dnf` | Servidores legados/minimal podem não ter `python3-dnf`. CLI direta funciona sempre. |
| Templates em **drop-in** (`/etc/ssh/sshd_config.d/`, `/etc/sudoers.d/`) | Preserva configurações existentes do host. Não substitui `sshd_config` nem `sudoers` principais. |
| `katello-ca-consumer` via base64 | Garante que o `.rpm` chegue íntegro mesmo em hosts onde o módulo `copy` poderia ter problemas. |
| `kinit` com `KRB5CCNAME` isolado | Evita conflito com ccache existente do usuário/serviço. |
| Restart sequencial `realmd → oddjobd → sshd → sssd` | Empiricamente resolve o caso de `realm list` retornar vazio após join. |
