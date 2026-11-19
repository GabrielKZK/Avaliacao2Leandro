# Respostas do desafio | Aula 13, Docker Compose

## 1. Por que só o frontend tem `ports`?

`ports` publica uma porta do contêiner no host, ou seja, abre uma porta para quem está fora do Docker. `expose` apenas documenta a porta para as outras redes do Compose, sem publicar nada.

| Serviço | Configuração | Motivo |
|---|---|---|
| frontend | `ports: "127.0.0.1:8080:80"` | É o único ponto de entrada. O navegador precisa alcançá-lo pelo host. |
| backend | `expose: "8000"` | Só o Nginx precisa chamá-lo, pelo nome `backend`, na rede `web_net`. |
| db | nenhuma | Só o backend precisa dele, pelo nome `db`, na rede `data_net`. |

Menos portas abertas significa menor superfície de ataque. Um acesso direto a `db:5432` ou `backend:8000` a partir do host falha, e isso é o comportamento esperado. Além disso, `data_net` é `internal: true`, então o banco nem tem rota de saída para a internet.

## 2. Evidência de persistência

A tabela é criada por `db/init.sql` (somente na primeira criação do volume) e os dados ficam no volume nomeado `pgdata`, que existe fora do ciclo de vida do contêiner. Para comprovar:

```sh
docker compose up -d --build
curl -X POST http://127.0.0.1:8080/api/tasks -H 'Content-Type: application/json' -d '{"titulo":"Persistência"}'
docker compose down          # remove contêineres e redes, mantém o volume
docker compose up -d
curl http://127.0.0.1:8080/api/tasks   # a tarefa "Persistência" continua na lista
docker volume ls               # mostra aula13_pgdata
```

Resultado esperado: a tarefa criada antes do `down` aparece depois do `up`. Se rodar `docker compose down -v`, o volume é apagado e a lista volta vazia.

## 3. Um risco de segurança e uma melhoria de confiabilidade

**Risco de segurança: credenciais em texto puro e API sem autenticação.**
A senha do banco vive no `.env` e chega ao backend por variável de ambiente (`DATABASE_URL`), visível em `docker inspect` e em `/proc` do contêiner. Além disso, qualquer pessoa que alcance a porta 8080 pode listar, criar e apagar tarefas, sem HTTPS nem limite de requisições.
Mitigações: usar Docker secrets ou um cofre de segredos, trocar a senha didática, manter o `.env` fora do Git (já está no `.gitignore`), e pôr autenticação, TLS e rate limiting em um proxy reverso antes de expor fora da máquina local.

**Melhoria de confiabilidade: reconexão automática ao banco e backup do volume.**
O `depends_on` com `service_healthy` só controla a ordem de inicialização; ele não reinicia nada quando o banco cai depois. Por isso o backend abre uma conexão nova a cada requisição: quando o banco volta, a API volta sozinha, sem reiniciar o contêiner. Foi testado: com o banco parado a API responde 503, e ao religar o banco responde 200 sem reinício do backend. Como complemento, vale agendar `pg_dump` periódico do banco, porque o volume protege contra recriação de contêiner, mas não contra corrupção ou exclusão acidental do próprio volume.

## Quiz de revisão (gabarito)

1. **F.** O Compose não oferece alta disponibilidade multi-host por conta própria.
2. **F.** `depends_on` simples só garante que o contêiner iniciou; para garantir que aceita SQL é preciso `condition: service_healthy`.
3. **V.** O DNS interno resolve o nome do serviço dentro da rede Compose.
4. **V.** `down` preserva volumes nomeados; só `down -v` os remove.
5. **F.** Tráfego interno entre serviços da mesma rede dispensa publicar portas no host.

## Extra implementado (opcional do item 4)

Rota `DELETE /api/tasks/<id>` com SQL parametrizado (`DELETE FROM tarefas WHERE id = %s`), retornando 204 em caso de sucesso e 404 se o id não existir, mais o botão "Excluir" no frontend.

## Entregáveis que dependem da sua máquina

Estes itens precisam do Docker rodando no seu computador, então não foi possível gerar aqui:

- [ ] Screenshot do `docker compose ps` com os três serviços `healthy`.
- [ ] Screenshot do navegador em http://127.0.0.1:8080 com uma tarefa criada.
- [ ] Screenshot da evidência de persistência (tarefa visível depois de `down` e `up -d`).
