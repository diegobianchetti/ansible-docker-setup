# ansible-docker-setup

Playbook Ansible para instalação e padronização do Docker CE em hosts Ubuntu 24.04.

## O que faz

- Adiciona o repositório oficial `download.docker.com` com chave GPG verificada
- Instala `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin` e `docker-compose-plugin`
- Implanta `/etc/docker/daemon.json` via template Jinja2, com `storage-driver`, `live-restore` e `log-driver` configuráveis
- Valida o JSON do `daemon.json` antes de aplicar (`validate:`)
- Confirma no `post_task` que o storage driver ativo bate com o esperado
- Suporta pin de versão com `dpkg hold` para evitar upgrades não planejados

## Pré-requisitos

| Requisito | Versão mínima |
|-----------|--------------|
| Ansible | 2.14 |
| Python (control node) | 3.9 |
| SO alvo | Ubuntu 24.04 (Noble) |
| Acesso SSH | usuário com `sudo` sem senha, ou senha via `--ask-become-pass` |

## Estrutura

```
ansible-docker-setup/
├── ansible.cfg
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       └── docker_hosts.yml      # variáveis com defaults comentados
├── playbooks/
│   └── docker-setup.yml          # ponto de entrada
└── roles/
    └── docker/
        ├── defaults/main.yml
        ├── handlers/main.yml
        ├── meta/main.yml
        ├── tasks/
        │   ├── main.yml
        │   ├── prereqs.yml       # GPG + repositório
        │   ├── install.yml       # pacotes + hold opcional
        │   ├── configure.yml     # daemon.json
        │   ├── service.yml       # systemd
        │   └── users.yml         # grupo docker
        └── templates/
            └── daemon.json.j2
```

## Uso rápido

```bash
# Clonar e entrar no repositório
git clone https://github.com/diego-oogway/ansible-docker-setup.git
cd ansible-docker-setup

# Ajustar inventory/hosts.yml com seus hosts reais

# Instalar coleções necessárias
ansible-galaxy collection install ansible.builtin

# Executar
ansible-playbook playbooks/docker-setup.yml

# Apenas configurar o daemon (hosts já com Docker instalado)
ansible-playbook playbooks/docker-setup.yml --tags configure
```

## Variáveis principais

Todas as variáveis com seus defaults estão em `roles/docker/defaults/main.yml`.
Para sobrescrever por grupo, edite `inventory/group_vars/docker_hosts.yml`.

| Variável | Default | Descrição |
|----------|---------|-----------|
| `docker_version` | `latest` | Versão do Docker CE. Ex.: `5:26.1.4-1~ubuntu.24.04~noble` |
| `docker_storage_driver` | `overlay2` | Storage driver — ver seção abaixo |
| `docker_live_restore` | `true` | Containers sobrevivem ao restart do daemon |
| `docker_log_driver` | `json-file` | Log driver padrão |
| `docker_log_opts` | `max-size: 100m, max-file: 5` | Rotação de logs |
| `docker_users` | `[]` | Lista de usuários a adicionar ao grupo `docker` |

## Decisões técnicas

### Por que `storage-driver: overlay2` explícito no daemon.json?

A partir do **Docker CE 26** (lançado em 2024) e consolidado no **Docker 29+**, o Docker passou a
suportar um segundo driver de overlayfs chamado `overlayfs` (sem o "2"), que usa montagem nativa
via syscall `mount(2)` com `fstype=overlay` diretamente, sem a camada de redirecionamento de
diretório introduzida pelo `overlay2` clássico.

O problema concreto é o seguinte:

1. **Detecção automática pode variar entre hosts.** Em kernels >= 5.13 com suporte a
   *idmapped mounts*, o Docker Engine detecta suporte nativo e pode preferir o driver novo.
   Em ambientes com kernels mistos (upgrade parcial de frota, VMs com kernels diferentes),
   hosts rodando o mesmo binário do Docker podem terminar com drivers distintos — o que gera
   inconsistência operacional e dificulta troubleshooting.

2. **`overlay2` e `overlayfs` armazenam layers em layouts ligeiramente diferentes.**
   Uma vez que o daemon inicia com um driver, os dados em `/var/lib/docker` ficam atrelados a
   ele. Trocar de driver exige `docker system prune` completo ou migração manual das imagens.
   Numa frota onde parte dos nós ainda usa `overlay2` e outra parte migrou automaticamente para
   `overlayfs`, operações como rsync de `/var/lib/docker` entre hosts ou restore de backup
   manual deixam de funcionar de forma transparente.

3. **`native-overlay-diff` e comportamento de diff de layers.** O `overlay2` com o kernel
   moderno pode ativar `native-overlay-diff` automaticamente (opção introduzida no Docker 19.03,
   mas com comportamento revisado no 26+). Isso altera como o Docker calcula o tamanho das
   layers em `docker image inspect` e `docker system df`. Fixar o driver impede que o
   comportamento mude silenciosamente após um upgrade do pacote.

**Decisão:** definir `"storage-driver": "overlay2"` explicitamente no `daemon.json` garante
comportamento determinístico e idêntico em toda a frota, independente da versão do kernel ou de
futuros defaults do Docker Engine. O custo é zero; o benefício é previsibilidade.

> **Nota para ambientes rootless:** em instalações rootless, o driver padrão é `fuse-overlayfs`
> e esta configuração não se aplica. Este playbook cobre exclusivamente instalações root (daemon
> gerenciado pelo systemd como root).

---

## Licença

MIT
