# HOWTO - Instalação do Traefik via Portainer Stack

Este documento descreve como instalar o **Traefik Proxy** usando **Portainer Stacks**, com dashboard protegido por Basic Auth, redirecionamento HTTP para HTTPS, certificado self-signed e um serviço de teste `whoami`.

O exemplo foi ajustado para acesso via IP:

```text
192.168.50.2
```

---

## 1. Objetivo

Ao final da instalação, o ambiente terá:

- Traefik rodando como proxy reverso.
- Entrada HTTP na porta `80`.
- Entrada HTTPS na porta `443`.
- Redirecionamento automático de HTTP para HTTPS.
- Dashboard do Traefik protegido por usuário e senha.
- Serviço `whoami` para teste.
- Rede Docker externa chamada `proxy`.
- Certificado self-signed para ambiente local/laboratório.

---

## 2. Pré-requisitos

Antes de iniciar, confirme que o servidor possui:

- Docker instalado.
- Portainer instalado e acessível.
- Acesso ao terminal do servidor.
- Permissão para criar redes Docker.
- Permissão para criar diretórios em `/opt`.
- Pacote `apache2-utils` ou equivalente para usar o comando `htpasswd`.
- `openssl` instalado.

### Instalar dependências em Debian/Ubuntu

```bash
sudo apt update
sudo apt install -y apache2-utils openssl
```

---

## 3. Criar a rede Docker

O Traefik e os containers que serão expostos por ele precisam estar na mesma rede Docker.

Execute no servidor:

```bash
docker network create proxy
```

Se a rede já existir, o Docker exibirá uma mensagem de erro informando que ela já existe. Nesse caso, pode seguir normalmente.

Para validar:

```bash
docker network ls
```

Confirme se existe uma rede chamada:

```text
proxy
```

---

## 4. Criar diretórios no servidor

Crie os diretórios que serão usados para armazenar o certificado e a configuração dinâmica do Traefik:

```bash
sudo mkdir -p /opt/traefik/certs
sudo mkdir -p /opt/traefik/dynamic
```

---

## 5. Criar certificado self-signed

Para ambiente local ou laboratório, pode ser usado um certificado self-signed.

Execute:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /opt/traefik/certs/local.key \
  -out /opt/traefik/certs/local.crt \
  -subj "/CN=192.168.50.2"
```

> Observação: por ser self-signed, o navegador exibirá aviso de segurança ao acessar o Traefik via HTTPS.

---

## 6. Criar arquivo de configuração TLS

Crie o arquivo:

```bash
sudo nano /opt/traefik/dynamic/tls.yaml
```

Conteúdo:

```yaml
tls:
  certificates:
    - certFile: /certs/local.crt
      keyFile: /certs/local.key
```

Salve o arquivo.

---

## 7. Gerar usuário e senha do dashboard

O dashboard do Traefik será protegido com Basic Auth.

Exemplo usando usuário `admin` e senha `admin`:

```bash
htpasswd -nb admin 'admin' | sed -e 's/\$/\$\$/g'
```

A saída será parecida com:

```text
admin:$$apr1$$xxxxxxxx$$yyyyyyyyyyyyyyyyyyyyyy
```

Copie a linha inteira.

> Importante: no compose usado pelo Portainer, o caractere `$` precisa estar duplicado como `$$`.  
> Por isso usamos o `sed -e 's/\$/\$\$/g'`.

---

## 8. Criar a Stack no Portainer

No Portainer:

1. Acesse **Stacks**.
2. Clique em **Add stack**.
3. Defina um nome, por exemplo:

```text
traefik
```

4. Cole o compose abaixo no editor.
5. Substitua o trecho do hash do Basic Auth pelo hash gerado no passo anterior.
6. Clique em **Deploy the stack**.

---

## 9. Docker Compose para Portainer

> Atenção: substitua a linha do Basic Auth pelo hash gerado no seu ambiente.

```yaml
services:
  traefik:
    image: traefik:v3.7
    container_name: traefik
    restart: unless-stopped

    security_opt:
      - no-new-privileges:true

    networks:
      - proxy

    ports:
      - "80:80"
      - "443:443"

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /opt/traefik/certs:/certs:ro
      - /opt/traefik/dynamic:/dynamic:ro

    command:
      # EntryPoints
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.web.http.redirections.entrypoint.permanent=true"

      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.websecure.http.tls=true"

      # Dynamic file provider
      - "--providers.file.filename=/dynamic/tls.yaml"

      # Docker provider
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=proxy"

      # API / Dashboard
      - "--api.dashboard=true"
      - "--api.insecure=false"

      # Logs / Metrics
      - "--log.level=INFO"
      - "--accesslog=true"
      - "--metrics.prometheus=true"

    labels:
      - "traefik.enable=true"

      # Dashboard acessando por IP
      - "traefik.http.routers.dashboard.rule=Host(`192.168.50.2`)"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.tls=true"

      # Basic Auth
      - "traefik.http.middlewares.dashboard-auth.basicauth.users=admin:$$apr1$$COLE_AQUI_O_RESTANTE_DO_HASH"
      - "traefik.http.routers.dashboard.middlewares=dashboard-auth@docker"

  whoami:
    image: traefik/whoami
    container_name: whoami
    restart: unless-stopped

    networks:
      - proxy

    labels:
      - "traefik.enable=true"

      # Whoami acessando pelo mesmo IP, mas com path /whoami
      - "traefik.http.routers.whoami.rule=Host(`192.168.50.2`) && PathPrefix(`/whoami`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls=true"
      - "traefik.http.services.whoami.loadbalancer.server.port=80"

networks:
  proxy:
    external: true
```

---

## 10. Acessos

Após o deploy, acesse:

### Dashboard Traefik

```text
https://192.168.50.2
```

Usuário e senha conforme gerado no passo de Basic Auth.

Exemplo:

```text
Usuário: admin
Senha: admin
```

### Serviço de teste whoami

```text
https://192.168.50.2/whoami
```

---

## 11. Validações após o deploy

### Verificar se os containers estão rodando

```bash
docker ps
```

Verifique se aparecem os containers:

```text
traefik
whoami
```

### Verificar logs do Traefik

```bash
docker logs traefik
```

### Testar HTTP para HTTPS

```bash
curl -I http://192.168.50.2
```

A resposta esperada é um redirecionamento para HTTPS.

### Testar HTTPS ignorando o certificado self-signed

```bash
curl -k https://192.168.50.2
```

### Testar o whoami

```bash
curl -k https://192.168.50.2/whoami
```

---

## 12. Observações importantes

### Sobre o uso de IP

Neste HOWTO foi usado o IP `192.168.50.2` diretamente na regra:

```yaml
Host(`192.168.50.2`)
```

Isso permite acessar o Traefik sem configurar DNS.

Para produção, o ideal é usar um domínio real, por exemplo:

```yaml
Host(`traefik.seudominio.com.br`)
```

E apontar o DNS para o IP do servidor.

---

### Sobre o certificado

O certificado criado neste HOWTO é self-signed.

Isso significa que:

- O HTTPS funciona.
- O navegador exibirá alerta de segurança.
- O certificado não é confiável publicamente.
- Para produção, recomenda-se usar Let's Encrypt ou certificado emitido por uma CA confiável.

---

### Sobre o Docker socket

O volume abaixo permite que o Traefik leia os containers e labels do Docker:

```yaml
- /var/run/docker.sock:/var/run/docker.sock:ro
```

Ele está montado como somente leitura (`ro`), o que é recomendado.

---

### Sobre a rede proxy

Todo container que deve ser exposto pelo Traefik precisa estar conectado à rede:

```yaml
networks:
  - proxy
```

E a stack deve declarar:

```yaml
networks:
  proxy:
    external: true
```

---

## 13. Troubleshooting

### Erro: 404 page not found

#### Causa provável

O Traefik recebeu a requisição, mas nenhum router correspondeu ao `Host` ou `Path`.

#### Verificações

Confirme se está acessando exatamente:

```text
https://192.168.50.2
```

Para o dashboard, a regra é:

```yaml
Host(`192.168.50.2`)
```

Para o whoami, a regra é:

```yaml
Host(`192.168.50.2`) && PathPrefix(`/whoami`)
```

Portanto, o whoami deve ser acessado em:

```text
https://192.168.50.2/whoami
```

#### Possíveis soluções

- Conferir se o IP no compose está correto.
- Conferir se as labels estão com indentação correta.
- Conferir se o container está na rede `proxy`.
- Conferir se o Docker provider está habilitado.
- Conferir os logs do Traefik.

Comando útil:

```bash
docker logs traefik
```

---

### Erro: senha incorreta no dashboard

#### Causa provável

O hash do Basic Auth foi gerado ou colado incorretamente.

#### Possíveis problemas

- O hash foi copiado incompleto.
- O `$` não foi duplicado como `$$`.
- A senha digitada não corresponde à senha usada no `htpasswd`.
- O navegador salvou credenciais antigas.

#### Solução

Gerar novamente:

```bash
htpasswd -nb admin 'admin' | sed -e 's/\$/\$\$/g'
```

Copiar a saída completa e substituir no compose:

```yaml
- "traefik.http.middlewares.dashboard-auth.basicauth.users=admin:$$apr1$$..."
```

Depois, redeploy da stack.

Se ainda falhar, teste em janela anônima do navegador.

---

### Erro: network proxy declared as external, but could not be found

#### Causa provável

A rede `proxy` não foi criada antes do deploy.

#### Solução

Execute no servidor:

```bash
docker network create proxy
```

Depois faça redeploy da stack no Portainer.

---

### Erro: porta 80 ou 443 já está em uso

#### Causa provável

Outro serviço já está usando as portas `80` ou `443`.

#### Verificação

```bash
sudo ss -tulpn | grep -E ':80|:443'
```

#### Solução

- Parar o serviço que está usando a porta.
- Alterar as portas publicadas no compose.
- Remover outro proxy reverso já existente, se for o caso.

---

### Erro: navegador mostra certificado inseguro

#### Causa provável

O certificado usado é self-signed.

#### Solução

Para ambiente de teste, aceitar o risco no navegador.

Para produção, usar certificado válido, preferencialmente com Let's Encrypt.

---

### Erro: whoami não abre

#### Causa provável

O serviço não está na rede `proxy`, a label está incorreta ou o path usado está errado.

#### Verificações

A URL correta é:

```text
https://192.168.50.2/whoami
```

O serviço precisa conter:

```yaml
networks:
  - proxy
```

E a label:

```yaml
- "traefik.http.routers.whoami.rule=Host(`192.168.50.2`) && PathPrefix(`/whoami`)"
```

---

### Erro: Traefik não detecta containers

#### Causa provável

Problema no Docker socket ou no Docker provider.

#### Verificações

Confirme se existe o volume:

```yaml
- /var/run/docker.sock:/var/run/docker.sock:ro
```

Confirme se o Docker provider está habilitado:

```yaml
- "--providers.docker=true"
```

Confirme se os containers possuem:

```yaml
- "traefik.enable=true"
```

---

## 14. Expondo uma nova aplicação pelo Traefik

Exemplo para expor um container Nginx em `/app`:

```yaml
services:
  app:
    image: nginx:alpine
    restart: unless-stopped

    networks:
      - proxy

    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`192.168.50.2`) && PathPrefix(`/app`)"
      - "traefik.http.routers.app.entrypoints=websecure"
      - "traefik.http.routers.app.tls=true"
      - "traefik.http.services.app.loadbalancer.server.port=80"

networks:
  proxy:
    external: true
```

Acesso:

```text
https://192.168.50.2/app
```

> Observação: algumas aplicações não funcionam corretamente em subpath como `/app` sem configuração adicional. Nesses casos, é melhor usar domínio ou subdomínio.

---

## 15. Recomendações para produção

Para ambiente produtivo, recomenda-se:

- Usar domínio real em vez de IP.
- Usar certificado Let's Encrypt.
- Criar senhas fortes para o dashboard.
- Restringir acesso ao dashboard por IP ou VPN.
- Não expor serviços sem autenticação quando forem administrativos.
- Manter o Traefik atualizado.
- Monitorar logs de acesso.
- Evitar usar `api.insecure=true`.
- Manter `providers.docker.exposedbydefault=false`.

---

## 16. Checklist rápido

Antes de pedir suporte, valide:

- [ ] A rede `proxy` existe.
- [ ] Os diretórios `/opt/traefik/certs` e `/opt/traefik/dynamic` existem.
- [ ] O arquivo `/opt/traefik/dynamic/tls.yaml` existe.
- [ ] O certificado `local.crt` existe.
- [ ] A chave `local.key` existe.
- [ ] O hash do Basic Auth foi colado completo.
- [ ] O hash usa `$$` no compose.
- [ ] O Traefik está rodando.
- [ ] O whoami está rodando.
- [ ] As portas `80` e `443` estão livres no host.
- [ ] O acesso está sendo feito por `https://192.168.50.2`.
- [ ] O whoami está sendo acessado por `https://192.168.50.2/whoami`.

---

## 17. Comandos úteis

```bash
docker ps
```

```bash
docker logs traefik
```

```bash
docker network ls
```

```bash
docker network inspect proxy
```

```bash
curl -k https://192.168.50.2
```

```bash
curl -k https://192.168.50.2/whoami
```

```bash
sudo ss -tulpn | grep -E ':80|:443'
```
