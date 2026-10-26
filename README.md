# Nextcloud Talk - Backend de Alto Desempenho (HPB)

## Visão Geral

Este guia explica como configurar um Backend de Alto Desempenho (HPB) para o Nextcloud Talk usando contêineres Docker. O HPB melhora significativamente a performance de videoconferências com múltiplos participantes.

**Imagem Janus mantida pela LibreCode:** [`ghcr.io/librecodecoop/janus` no GHCR](https://github.com/LibreCodeCoop/janus-docker/pkgs/container/janus), com [Dockerfile de origem](https://github.com/LibreCodeCoop/janus-docker/blob/main/Dockerfile) e [releases versionadas](https://github.com/LibreCodeCoop/janus-docker/releases). Para produção, use uma tag aprovada com digest OCI, não `latest`.


### Ansible

Automação de exemplo para deploy do HPB. Adapte a playbook ao seu ambiente e às suas credenciais.
### Arquitetura do sistema

O HPB consiste em três componentes principais:

1. **Serviço STUN/TURN (Eturnal)** - Gerencia conectividade através de NAT/firewall
2. **Serviço de Sinalização (spreedbackend)** - Coordena comunicação WebRTC entre participantes
3. **Servidor Janus** - Processa streams de áudio e vídeo


---

## Instalação

Este guia cobre a instalação por subdomínio. Se você preferir usar um caminho (path), adapte a configuração do proxy reverso ao seu ambiente.

### Pré-requisitos

### DNS
Configure as seguintes entradas no seu provedor de DNS:
- `turn.example.com`
- `talk-signaling.example.com`
- `janus.example.com`



### Aplicativos Nextcloud
Instale o aplicativo `talk` na sua instância Nextcloud.

### Portas Necessárias
Certifique-se de que as seguintes portas estejam liberadas no firewall:
```
3478/tcp/udp       # TURN/STUN
5349/tcp           # TURN com TLS (opcional)
50000-50100/udp    # Portas de relay do TURN
```

Liberando as portas no UFW:
```
ufw allow 3478/tcp
ufw allow 3478/udp
ufw allow 5349/tcp
ufw allow 50000:50100/udp
ufw allow 8080,8443,8188/tcp
```

---

## Etapa 1: Geração de segredos

O HPB requer três segredos únicos para autenticação:

### Gerando segredos
```bash
# Gere 3 segredos diferentes usando openssl
openssl rand --hex 32
```

Execute o comando 3 vezes para obter:
- **TURN_SECRET**: Autenticação do servidor TURN
- **SIGNALING_SECRET**: Autenticação do serviço de sinalização  
- **INTERNAL_SECRET**: Comunicação interna entre componentes

### Exemplo de segredos
```bash
TURN_SECRET=86a057a959534eb761b20afb4122da6b5475a5f8db20ba16abbb91c65132ca03
SIGNALING_SECRET=f7d8e9a0b1c2d3e4f5g6h7i8j9k0l1m2n3o4p5q6r7s8t9u0v1w2x3y4z5a6b7c8
INTERNAL_SECRET=a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6a7b8c9d0e1f2
```

⚠️ **IMPORTANTE**: Substitua os valores de exemplo pelos seus próprios segredos gerados.

---

## Etapa 2: Configuração do servidor TURN (Eturnal)

### Configuração do contêiner
```yaml
networks:
  reverse-proxy:
    external: true
    name: reverse-proxy
  internal:
    driver: bridge

services:
  eturnal:
    image: ghcr.io/processone/eturnal:latest
    container_name: eturnal-talk
    ports:
      - "3478:3478/udp"
      - "3478:3478/tcp"
      - "5349:5349/tcp"
      - "50000-50100:50000-50100/udp"
    restart: unless-stopped
    environment:
      - ETURNAL_SECRET=${TURN_SECRET}
      - ETURNAL_REALM=turn.example.com
      - ETURNAL_RELAY_IPV4_ADDR=SEU_IP_PUBLICO
      - ETURNAL_RELAY_MIN_PORT=50000
      - ETURNAL_RELAY_MAX_PORT=50100
      - VIRTUAL_HOST=turn.example.com
      - LETSENCRYPT_HOST=turn.example.com
      - LETSENCRYPT_EMAIL=email@example.com
      # Não há VIRTUAL_PORT aqui porque o proxy reverso não trabalha com UDP.
    volumes:
      - ./eturnal/eturnal.yml:/etc/eturnal.yml:ro
      - ./volumes/eturnal/logs:/var/log/eturnal
    networks:
      - internal
      - reverse-proxy
```

### Arquivo de configuração (`eturnal/eturnal.yml`)
```yaml
eturnal:
  secret: "${TURN_SECRET}"
  realm: "turn.example.com"
  relay_ipv4_addr: "SEU_IP_PUBLICO"
  relay_ipv6_addr: "SEU_IP_PUBLICO_V6" # Opcional
  relay_min_port: 50000
  relay_max_port: 50100
  strict_expiry: false
  log_dir: stdout
  listen:
    - ip: "::"
      port: 3478
      transport: udp
      enable_turn: true
    - ip: "::"
      port: 3478
      transport: tcp
      enable_turn: true
  modules:
    mod_log_stun: {}
```

Se você for usar `turns://` em produção, adicione também `tls_crt_file` e `tls_key_file` com um certificado válido.

### Teste do servidor TURN
Use a ferramenta online para testar:
- URL: https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/
- STUN/TURN URI: `turn:turn.example.com:3478`
- Username: `user-test`
- Password: `SEU_TURN_SECRET`


Clique em `Add server` e depois em `Gather Candidates`.
O IP e a porta devem ser listados.

---

## Etapa 3: Configuração do serviço de sinalização (`spreedbackend`)
### Arquivos de configuração
Crie o arquivo `volumes/hpb/config/server.conf` com o seguinte conteúdo (altere conforme necessário):

```bash
[http]
listen = 0.0.0.0:8080

[app]
debug = false
trustedproxies = 172.0.0.0/8

[sessions]
hashkey = ${SESSION_HASHKEY} # Substitua por uma string aleatória de 32 ou 64 bytes
blockkey = ${SESSION_BLOCKKEY} # Substitua por uma string aleatória de 16, 24 ou 32 bytes para criptografar a sessão

[clients]
internalsecret = ${INTERNAL_SECRET}

[backend]
backends = backend
allowall = false
timeout = 20
connectionsperhost = 40

[backend]
url = https://${NEXTCLOUD_URL} # URL da sua instância do Nextcloud

secret = ${SIGNALING_SECRET} # Segredo gerado anteriormente. Deve ser inserido no plugin do Talk.

[nats]
# Endereço do serviço NATS
url = nats://nats:4222

[mcu]
type = janus

# URL do serviço Janus
url = ws://janus:8188/

maxstreambitrate = 1048576
maxscreenbitrate = 2097152

[turn]
# Chave da API usada quando o MCU requisitar as credenciais do TURN
apikey = ${TURN_API_KEY}

# Segredo do TURN. Deve ser o mesmo configurado no servidor TURN.
secret = ${TURN_SECRET}

# Lista de servidores TURN a serem usados
servers = turn:turn.example.com:3478?transport=udp,turn:turn.example.com:3478?transport=tcp
```

Adicione o arquivo `volumes/hpb/config/gnatsd.conf` com a seguinte configuração:
```bash
cluster {

	port: 4244  # port for inbound route connections

	routes = [
		# You can add other servers here to build up a cluster.
		#nats-route://otherserver:4244
	]

}
```

### Configuração do contêiner do HPB

```yaml
  spreedbackend:
    image: mwalbeck/nextcloud-spreed-signaling
    volumes:
      - ./volumes/hpb/config/server.conf:/config/server.conf
      - ./volumes/hpb/ssl/:/etc/nginx/ssl/
    restart: unless-stopped
    environment:
      - JANUS_URL=ws://janus:8188/
      - USE_JANUS=1
      - VIRTUAL_HOST=talk-signaling.example.com
      - LETSENCRYPT_HOST=talk-signaling.example.com
      - LETSENCRYPT_EMAIL=email@example.com
      - VIRTUAL_PORT=8080
    networks:
      - reverse-proxy
      - internal
    extra_hosts:
      - "hpb.example.com:SEU_IP_PUBLICO" # Domínio e IP público

```

Repositório upstream: [nextcloud-spreed-signaling](https://github.com/strukturag/nextcloud-spreed-signaling)

### Teste do Serviço de Sinalização
```bash
curl -i https://talk-signaling.example.com/api/v1/welcome
```

**Resposta esperada:**
```json
{
  "nextcloud-spreed-signaling": "Welcome",
  "version": "1.3.2~docker"
}
```

---

## Etapa 4: Configuração do servidor Janus

A imagem independente do Janus é publicada no [GHCR da LibreCode](https://github.com/LibreCodeCoop/janus-docker/pkgs/container/janus); o código-fonte está em [LibreCodeCoop/janus-docker](https://github.com/LibreCodeCoop/janus-docker) e as versões aprovadas estão nas [releases](https://github.com/LibreCodeCoop/janus-docker/releases).

**Exemplo validado em 25/10/2026** (consulte as releases para obter uma referência mais recente):

```yaml
services:
  janus:
    image: ghcr.io/librecodecoop/janus:1.4.3-r1@sha256:7d9c1bd28bdbc7702de924ed8a676f5c1f11afa3abc888654d8d4fc8b78a00d2
    # A imagem já possui ENTRYPOINT /usr/local/bin/janus.
    command: ["--full-trickle", "--nat-1-1=${PUBLIC_IP:?Informe o IPv4 publico}"]
    networks:
      - internal
    ports:
      - "20000-20100:20000-20100/udp"
    restart: unless-stopped
    volumes:
      - ./volumes/janus/config/janus.jcfg:/usr/local/etc/janus/janus.jcfg:ro
      - ./volumes/janus/config/janus.transport.websockets.jcfg:/usr/local/etc/janus/janus.transport.websockets.jcfg:ro
      - ./volumes/janus/config/janus.eventhandler.wsevh.jcfg:/usr/local/etc/janus/janus.eventhandler.wsevh.jcfg:ro

networks:
  internal:
    driver: bridge
```

A rede `internal` deve ser compartilhada pelo Janus e pelo `spreedbackend`. Mantenha o WebSocket `8188/tcp` acessível somente nessa rede; a sinalização utiliza `ws://janus:8188/`.

Crie antes de executar o Compose:

```bash
mkdir -p volumes/janus/config
```

`volumes/janus/config/janus.jcfg`:

```text
media: {
  rtp_port_range = "20000-20100"
}
events: {
  broadcast = true
}
```

`volumes/janus/config/janus.transport.websockets.jcfg`:

```text
general: {
  ws = true
  ws_port = 8188
  wss = false
}
admin: {
  admin_ws = false
}
```

`volumes/janus/config/janus.eventhandler.wsevh.jcfg`:

```text
general: {
  enabled = true
  events = "handles,media,webrtc"
  grouping = true
  backend = "ws://spreedbackend:8080/spreed"
  subprotocol = "janus-events"
  events_cap_on_reconnect = 500
  events_max_queue = 2000
}
```

**Importante:** use o arquivo real de eventos `janus.eventhandler.wsevh.jcfg`, não o antigo `janus.eventhandler.sampleevh.jcfg`. Não monte um arquivo inexistente: o Docker pode criar um diretório no host. A imagem nova já inclui os módulos VideoRoom, WebSocket e `wsevh`, sem necessidade de compilação local.

Valide com `docker compose config --quiet` e `docker compose logs --tail=100 janus`. Testes de conexão de sinalização não substituem áudio, vídeo, compartilhamento de tela e TURN entre usuários em redes distintas.

## Etapa 5: Configuração no Nextcloud

### Configuração do Nextcloud Talk
Acesse `/settings/admin/talk` no seu Nextcloud:

Clique em `Adicionar servidor de back-end de alto desempenho`.

Preencha com a **URL do servidor de alto desempenho** e o `SIGNALING_SECRET` criado anteriormente. No exemplo, use `wss://talk-signaling.example.com`.


**Servidor TURN:**
Adicione um novo servidor clicando em `Adicionar um novo servidor TURN`.

- Tipo de TURN: `turn` e `turns`
- URL do servidor TURN: `turn.example.com:3478`
- Segredo do servidor TURN: `SEU_TURN_SECRET`
- Protocolos: TCP e UDP habilitados

**Servidor TURN com TLS:**
- URL do servidor TURN com TLS: `turns.example.com:5349`
- Segredo: `SEU_TURN_SECRET`

### Configuração do HPB
**Infraestrutura de Alto Desempenho:**
- URL: `https://talk-signaling.example.com`
- Segredo compartilhado: `SEU_SIGNALING_SECRET`

---

## Solução de problemas

### Problemas comuns

#### 1. Erro de Autorização (401)
**Sintoma:**
```
session [...]: realm <hpb.example.com> user <>: incoming packet message processed, error 401: Unauthorized
```

**Solução:**
- Verifique se o segredo no frontend (Nextcloud) é idêntico ao configurado no backend (Eturnal)
- Confirme que não há espaços extras nos segredos

#### 2. Timeout de Conexão
**Sintoma:**
```
Could not send request [...]: context deadline exceeded
```

**Solução:**
- Verifique conectividade de rede entre componentes
- Confirme que as portas estão abertas no firewall
- Verifique se os DNS estão resolvendo corretamente

#### 3. Erro NATS
**Sintoma:**
```
Could not create connection (nats: no servers available for connection)
```

**Solução:**
- Verifique conflitos de nome de contêiner
- Use nomes únicos para os contêineres
- Reinicie os serviços na ordem correta

#### 4. Warnings de SSRC
**Sintoma:**
```
Unknown SSRC, dropping packet (SSRC ...)
```

**Explicação:**
- Isso é normal durante videoconferências
- Indica múltiplos streams de áudio/vídeo sendo processados

---

## Monitoramento e logs

### Comandos úteis
```bash
# Verificar logs do Eturnal
docker compose logs eturnal -f

# Verificar logs do Signaling
docker compose logs spreedbackend -f

# Verificar logs do Janus
docker compose logs janus -f

# Verificar status dos contêineres
docker compose ps
```

### Ferramentas de diagnóstico

#### Firefox WebRTC
- Acesse: `about:webrtc`
- Monitore conexões ativas e estatísticas

#### Teste de Conectividade
```bash
# Teste TURN
curl -v turn.example.com:3478

# Teste Signaling
curl -v https://talk-signaling.example.com/api/v1/welcome

# Teste Janus
# Se a API HTTP estiver exposta, o endpoint padrão é:
curl -v http://janus.example.com:8088/janus/info
```

---

## Referências

### Documentação oficial
- [Nextcloud Talk](https://nextcloud.com/talk/)
- [Eturnal Documentation](https://eturnal.net/doc/)
- [Janus Gateway](https://janus.conf.meetecho.com/)

### Repositórios
- [Nextcloud Docker Images](https://github.com/nextcloud/docker)
- [Spreed Signaling](https://github.com/mwalbeck/docker-nextcloud-spreed-signaling)
- [Eturnal](https://github.com/processone/eturnal)

### Ferramentas de teste
- [WebRTC Trickle ICE](https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/)
- [TURN Server Tester](https://gabrieltanner.org/blog/turn-server/)

---
