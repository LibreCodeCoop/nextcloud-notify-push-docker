# nextcloud-notify-push-docker

Empacotamento Docker do `notify_push`, com o codigo-fonte upstream versionado em `./notify_push` via `git subtree`.

Este repositorio adiciona:

- um README em portugues para o projeto `notify_push` vendorizado
- um exemplo de deploy atras do `nginx-proxy`
- a configuracao de GitHub Actions para publicar a imagem no GHCR

## Estrutura do repositorio

- `notify_push/`: arvore de codigo do `nextcloud/notify_push`
- `docker-compose.example.yml`: exemplo de deploy com `nginx-proxy`
- `.github/workflows/publish-ghcr.yml`: workflow de publicacao da imagem no GHCR

## Deploy atras do nginx-proxy

Use o mesmo host para o Nextcloud e para o `notify_push`, roteando `/push/` para o servico de push.

Exemplo usando o dominio generico `cloud.example.com`:

```yaml
services:
  nextcloud:
    image: nextcloud:apache
    environment:
      VIRTUAL_HOST: cloud.example.com
      VIRTUAL_PATH: /
    networks:
      - reverse-proxy
      - internal

  notify_push:
    image: ghcr.io/librecodecoop/nextcloud-notify-push-docker:latest
    environment:
      PORT: "7867"
      NEXTCLOUD_URL: "https://cloud.example.com"
      REDIS_URL: "redis://redis:6379/0"
      DATABASE_URL: "postgres://nextcloud:secret@postgres/nextcloud"
      DATABASE_PREFIX: ""
      LOG: "info"
      VIRTUAL_HOST: cloud.example.com
      VIRTUAL_PATH: /push/
      VIRTUAL_DEST: /
      VIRTUAL_PORT: "7867"
    expose:
      - "7867"
    networks:
      - reverse-proxy
      - internal
```

Pontos importantes:

- o `notify_push` precisa estar na mesma rede Docker do `nginx-proxy`
- use `VIRTUAL_PATH=/push/` com barra final
- use `VIRTUAL_DEST=/` para remover o prefixo `/push/` antes de encaminhar a requisicao
- em setups com `nginx-proxy`, normalmente `expose` e suficiente e voce nao precisa publicar a porta `7867` no host

## Configuracao no Nextcloud

Depois que o servico estiver acessivel pelo proxy reverso:

```bash
occ app:enable notify_push
occ notify_push:setup https://cloud.example.com/push
occ notify_push:self-test
```

## Como testar

Valide as respostas do proxy:

```bash
curl -I https://cloud.example.com/push
curl -I https://cloud.example.com/push/
```

O comportamento esperado e:

- `/push` responder com `301` para `/push/`
- `/push/` nao responder com `502` ou outro erro de upstream do proxy

Enquanto isso, acompanhe os logs do servico e gere um evento no Nextcloud, como uma notificacao, mensagem no Talk ou
alteracao de arquivo:

```bash
docker compose logs -f notify_push
```

## Publicacao no GHCR

O workflow em `.github/workflows/publish-ghcr.yml` constroi a imagem a partir de `notify_push/Dockerfile` e publica no
GitHub Container Registry.

Ele executa em:

- pushes para `main`
- tags de versao no formato `v*`
- disparo manual

Requisitos do repositorio:

- GitHub Actions habilitado
- permissao `packages: write` para o workflow

O nome final da imagem e derivado do caminho do repositorio, por exemplo:

```text
ghcr.io/librecodecoop/nextcloud-notify-push-docker:latest
ghcr.io/librecodecoop/nextcloud-notify-push-docker:main
```

## Documentacao em portugues do upstream

O README em portugues do projeto vendorizado esta em
[`notify_push/README.pt-BR.md`](/home/mohr/git/librecode/nextcloud-notify-push-docker/notify_push/README.pt-BR.md).
