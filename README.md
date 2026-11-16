# PostgreSQL 17 em Ubuntu com Ansible

Requer controlador com Python 3 e Ansible Core 2.18/2.19; servidor Ubuntu 22.04 ou posterior suportado pelo PGDG, systemd, SSH com sudo e pelo menos 4 GiB de RAM. O servidor precisa acessar os repositórios Ubuntu, `www.postgresql.org` e `apt.postgresql.org` por HTTPS. Nenhum servidor é alterado até executar o playbook com um inventário real.

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install 'ansible==12.3.0'
# Esse pacote inclui community.postgresql 4.2.0.
# Alternativa: instalar ansible-core e executar ansible-galaxy collection install -r requirements.yml.
cp inventory/hosts.ini.example inventory/hosts.ini
# Edite host, usuário e parâmetros SSH no inventário.
ansible-playbook postgresql.yml --syntax-check
ansible-playbook postgresql.yml --ask-become-pass
```

O arquivo `ansible.cfg` também consulta a coleção preparada em `/workspace/.ansible/collections` neste ambiente. Fora dele, a instalação padrão da coleção no home é suficiente.

## O que configura

- PGDG com assinatura APT e TLS verificados; instala/atualiza exclusivamente a série 17 para a revisão disponível no repositório. Reexecutar pode atualizar pacotes e reiniciar o banco: agende janela de manutenção.
- Locale `pt_BR.UTF-8` (Português/Brasil com UTF-8), inclusive como `LANG` padrão para novas sessões. ISO-8859-1 é outra codificação e não é usada.
- Cluster `17/main`, checksums na criação, porta 5432, 500 conexões totais (algumas são reservadas pelo PostgreSQL para administração).
- Ajustes por RAM e vCPUs; `shared_buffers` usa 12,5% da RAM física, `work_mem` conservador e paralelismo por consulta desabilitado. `effective_cache_size` é uma estimativa, não uma alocação.
- Limite agregado `MemoryMax` de 50% de **MemTotal**, sem swap para o serviço, com pressão a partir de 90% desse orçamento. Inclui processos filhos e memória contabilizada pelo cgroup. Não reserva memória nem usa `MemAvailable`, que varia durante a execução. O limite não se aplica a outros clusters existentes.
- Base `key_cloack_db`, owner `key_cloack_owner` sem privilégios administrativos e senha aleatória de 48 bytes, persistida em `/var/lib/postgresql/key_cloack_db.credentials.json`, proprietário postgres, modo `0600`. A senha não aparece nas saídas do Ansible e é reutilizada nas execuções seguintes. Preserve esse arquivo: removê-lo gera nova senha e altera o usuário na execução seguinte.
- Inicialização, habilitação no boot e teste de autenticação TCP com o owner na base criada.

500 sessões não garantem 500 consultas pesadas simultâneas. `work_mem` pode ser consumido várias vezes por consulta. Sob pressão, o teto do cgroup pode causar OOM e interrupção do banco; faça testes com a carga real e considere pool de conexões. O mínimo de 4 GiB é uma barreira básica, não recomendação de capacidade.

## Acesso remoto

Por padrão escuta apenas em localhost. Para uma rede privada, use um arquivo de variáveis, por exemplo:

```yaml
pg_listen_addresses: 'localhost,10.0.0.10'
pg_client_cidrs:
  - 10.0.1.0/24
```

Execute com `-e @variaveis.yml`. As regras autorizam somente a base e o owner definidos; configure também firewall e TLS de acordo com sua rede. SCRAM autentica a senha, mas não cifra o tráfego. O playbook substitui `pg_hba.conf` do cluster escolhido: revise antes de aplicar em um servidor existente.

## Segurança e verificação

Não existe garantia de uma versão “sem CVEs”. Consulte os [avisos oficiais](https://www.postgresql.org/support/security/) e a revisão candidata de `postgresql-17` no PGDG no momento da implantação. `state: latest` busca a atualização disponível, mas não constitui uma auditoria de vulnerabilidades nem mantém atualizações automáticas após a execução.

```bash
sudo systemctl status postgresql@17-main
sudo systemctl show postgresql@17-main -p MemoryMax -p MemoryHigh -p MemorySwapMax
sudo -u postgres psql -p 5432 -d key_cloack_db -c 'SHOW server_version;'
sudo -u postgres psql -p 5432 -d key_cloack_db -c 'SHOW max_connections;'
sudo -u postgres psql -p 5432 -d key_cloack_db -c 'SHOW shared_buffers;'
```

Em clusters existentes, o playbook não recria dados nem muda o locale do cluster. A base usa `template0` com o locale solicitado; uma base existente com locale incompatível requer migração separada. Configurações em `postgresql.auto.conf` podem sobrepor arquivos gerenciados: a validação final detecta divergências nos parâmetros principais.

Validação feita no controlador: sintaxe com Ansible Core 2.19.15/community.postgresql 4.2.0 e renderização dos dois templates com orçamentos para 4, 16 e 64 GiB (9 tarefas aprovadas). Instalação, reinicialização, idempotência no servidor e carga de 500 conexões ainda precisam ser verificadas em um Ubuntu de destino.
