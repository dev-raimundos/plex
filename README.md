# Plex — Media Stack

> Stack Docker para download automático e streaming de filmes e séries via torrent, com interface de pedidos integrada ao Plex.

---

## O que é isso

Este repositório contém a configuração completa para subir uma **media stack de automação** via Docker Compose. A ideia é simples: o usuário acessa o **Overseerr**, pesquisa um filme ou série, clica em "Request" — e o conteúdo aparece automaticamente no **Plex** alguns minutos depois, já pronto para assistir.

Por baixo, o fluxo passa por quatro camadas: gerenciamento de pedidos (**Overseerr**), automação de mídia (**Radarr** para filmes e **Sonarr** para séries), busca em trackers (**Prowlarr**) e download efetivo (**qBittorrent**). O Plex detecta automaticamente os arquivos novos via monitoramento de pasta.

A stack foi construída para deploy via **Coolify**, usando volumes gerenciados pelo Docker e variáveis de ambiente para toda configuração sensível.

---

## Arquitetura

```
Usuário
   │
   │  Acessa Overseerr e pede um filme
   ▼
┌──────────────────────────────┐
│          Overseerr           │  ← UI de pedidos integrada ao Plex (:5055)
└──────────────┬───────────────┘
               │ envia pedido
               ▼
┌──────────────┴───────────────┐     ┌──────────────────────────┐
│           Radarr             │     │          Sonarr           │
│        (filmes :7878)        │     │       (séries :8989)      │
└──────────────┬───────────────┘     └────────────┬─────────────┘
               │                                  │
               │  busca torrent ideal             │
               ▼                                  ▼
         ┌─────────────────────────────────────────┐
         │               Prowlarr                  │  ← indexador de trackers (:9696)
         └──────────────────────┬──────────────────┘
                                │ envia magnet/torrent
                                ▼
                  ┌─────────────────────────┐
                  │       qBittorrent       │  ← client torrent (:8080)
                  └─────────────┬───────────┘
                                │ arquivo baixado em /downloads
                                │ Radarr/Sonarr move para /movies ou /tv
                                ▼
               ┌────────────────────────────────┐
               │             Plex               │  ← streaming (:32400)
               │   detecta arquivo novo         │
               │   biblioteca atualizada        │
               └────────────────────────────────┘
```

Todos os containers se comunicam por uma rede bridge isolada (`internal`). Nenhum serviço interno expõe portas desnecessárias para o host além das necessárias para o Coolify rotear o tráfego.

---

## Engenharia das decisões

### Imagens LinuxServer.io (`lscr.io/linuxserver/*`)

Todas as imagens usam o repositório do [LinuxServer.io](https://linuxserver.io) ao invés das imagens oficiais de cada projeto ou de terceiros. O LinuxServer mantém builds atualizados, documentação consistente e — mais importante — suporte nativo ao padrão **PUID/PGID**.

### PUID e PGID

Containers Docker rodam internamente como root por padrão. Sem ajuste, os arquivos criados nos volumes pertencem ao root no host, causando problemas de permissão ao acessar a mídia de fora do container. Com `PUID` e `PGID` apontando para o usuário real do servidor, todos os arquivos gerados pelos containers (configs, mídia, downloads) pertencem ao usuário correto desde o início.

### Prowlarr ao invés de Jackett

Jackett é o indexador mais antigo e ainda amplamente usado, mas exige configuração manual em cada aplicação (Radarr, Sonarr). O **Prowlarr** sincroniza os indexers automaticamente com todos os apps via API — adiciona um tracker uma vez, propaga para todos. É a escolha atual do ecossistema *arr.

### Overseerr como frontend

O Overseerr autentica usuários via conta Plex — quem já tem acesso ao servidor Plex consegue fazer pedidos sem criar uma nova conta. A interface é pensada para usuários não-técnicos: busca, clica, pede. Toda a complexidade do Radarr/Sonarr fica escondida.

### Volumes nomeados ao invés de bind mounts

Em um deploy via Coolify em VPS, bind mounts exigiriam gerenciar caminhos absolutos no servidor host, criar diretórios manualmente e lidar com permissões de forma mais frágil. Volumes nomeados do Docker são gerenciados pelo runtime e funcionam de forma previsível independente de onde o Coolify mantém os dados. O Coolify também lida bem com volumes nomeados no seu painel de administração.

### Volume `downloads` compartilhado

O qBittorrent baixa os arquivos no volume `/downloads`. Radarr e Sonarr têm acesso ao mesmo volume — quando o download termina, eles movem (hardlink ou copy) o arquivo para `/movies` ou `/tv`. Isso evita mover dados entre volumes diferentes (o que seria uma cópia real, lenta) e mantém tudo na mesma camada de storage do Docker.

### Porta 6881 para torrent

O qBittorrent precisa de uma porta de entrada para conexões peer-to-peer. A `6881` é a porta padrão da maioria dos clients e está aberta em TCP e UDP. Sem ela, o client opera apenas em modo de saída, reduzindo drasticamente a velocidade de download e upload para o swarm.

### Sem `network_mode: host`

A documentação do Plex sugere `network_mode: host` para descoberta local de rede (DLNA, GDM). Para um deploy em Coolify rodando atrás do Traefik, isso conflita com o gerenciamento de rede do proxy e pode expor portas indesejadas. O modo bridge com a porta `32400` exposta é suficiente para streaming remoto e compatível com o roteamento do Coolify.

### Variáveis de ambiente via `.env`

O `PLEX_CLAIM` é um token sensível. As variáveis ficam em `.env` separado do `compose.yaml` para permitir versionar o compose sem expor credenciais. O `.env.example` documenta o que é necessário sem valores reais.

---

## Estrutura do repositório

```
.
├── compose.yaml      # definição dos serviços
├── .env              # credenciais e configuração local (não versionar)
└── .env.example      # modelo sem valores reais (pode versionar)
```

Os volumes são gerenciados pelo Docker e ficam fora do repositório.

---

## Como usar

**1. Configure as variáveis de ambiente**

Copie o `.env.example` e preencha:

```bash
cp .env.example .env
```

```bash
# ID do usuário e grupo no servidor host
PUID=1000
PGID=1000

# Fuso horário
TZ=America/Sao_Paulo

# Token de vinculação do Plex — gere em https://www.plex.tv/claim/
# Atenção: expira em 4 minutos. Gere na hora do primeiro deploy.
PLEX_CLAIM=claim-XXXXXXXXXXXXXXXX
```

Para descobrir seu PUID e PGID, rode no servidor:

```bash
id -u && id -g
```

**2. Deploy via Coolify**

- Crie um novo serviço → **Docker Compose**
- Cole o conteúdo do `compose.yaml` ou aponte para o repositório
- Adicione as variáveis de ambiente no painel do Coolify
- Faça o deploy

**3. Configure os serviços (ordem importa)**

Após o deploy, acesse cada serviço pela porta correspondente e configure na seguinte ordem:

**Prowlarr** → `http://SEU-HOST:9696`
Adicione os indexers (trackers de torrent) que deseja usar. Depois vá em *Settings → Apps* e adicione Radarr e Sonarr para sincronização automática.

**Radarr** → `http://SEU-HOST:7878`
- *Settings → Indexers* → adicione Prowlarr como indexer
- *Settings → Download Clients* → adicione qBittorrent (host: `qbittorrent`, porta: `8080`)
- *Settings → Media Management* → Root Folder: `/movies`

**Sonarr** → `http://SEU-HOST:8989`
- Mesma configuração do Radarr, mas Root Folder: `/tv`

**Plex** → `http://SEU-HOST:32400/web`
- Crie as bibliotecas apontando para `/movies` e `/tv`
- O Plex detecta novos arquivos automaticamente quando Radarr/Sonarr finalizam um download

**Overseerr** → `http://SEU-HOST:5055`
- Faça login com sua conta Plex
- Conecte ao servidor Plex local
- Adicione Radarr e Sonarr como serviços de mídia
- A partir daqui, o Overseerr é a única interface que os usuários precisam usar

**4. Testando o fluxo completo**

No Overseerr, pesquise um filme e clique em *Request*. Acompanhe o download em `http://SEU-HOST:8080` (qBittorrent). Quando concluído, o arquivo aparece automaticamente na biblioteca do Plex.

---

## Requisitos

- Docker Engine 24+
- Docker Compose v2 (plugin, não o `docker-compose` legado)
- Coolify configurado no servidor de destino
- Conta no [plex.tv](https://www.plex.tv) para geração do claim token
- Portas `32400`, `5055`, `7878`, `8989`, `9696`, `8080` e `6881` disponíveis no host

---

## Notas de segurança

- O arquivo `.env` **nunca** deve ser commitado no Git — o `.gitignore` deve excluí-lo.
- As interfaces administrativas (Radarr, Sonarr, Prowlarr, qBittorrent) não têm autenticação forte por padrão — restrinja o acesso a elas por IP ou coloque atrás de VPN. Somente o Overseerr e o Plex devem ser expostos publicamente.
- O token `PLEX_CLAIM` é de uso único e expira em 4 minutos após geração. Após o primeiro boot bem-sucedido, ele não é mais necessário e pode ser removido da configuração.
- A porta `6881` do torrent deve ser aberta apenas se você quiser melhorar a velocidade de seed/download. Em ambientes corporativos ou redes com restrições, pode ser omitida.

---

*Stack pessoal de automação de mídia — clicou, baixou, assistiu.*
