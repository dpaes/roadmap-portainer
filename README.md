# Roadmap Portainer
  Descrição: Um passo a passo de como instalar o portainer para gerenciar containers de aplicações visualmente no seu VPS.

## Requisitos
  Você precisa ter uma VPS (Servidor Virtual Privado) e um Dominio para fazer posteriormente apontamento de DNS para o portainer.
  
  É possível seguir usando a própria máquina (localhost), porém em algumas aplicações apresentará problema por conta do certificado SSL.
  
  O VPS precisa ser linux (nesse roadmap uso o Ubuntu 20.04, mas fique a vontade para usar outra distribuição linux)

## Passo 1 - Instalando e Atualizando Recursos
  1.1 - Acessar a VPS pelo seu terminal (powerShell ou Bitvise ou Putty) via SSH.
  
  1.2 - Mudar para root ```sudo su``` e digitar sua senha root(se pedir).
  
  1.3 - Fazer um update e upgrade nos pacotes do linux ```apt-get update``` e ```apt-get upgrade -y```.
  
  1.4 - Baixar o Docker via Curl ```curl -fsSL https://get.docker.com -o get-docker.sh``` 
  > [!TIP]
  > caso não tenha Curl, baixe o pacote do curl usando ```apt-get install curl```.
  
  1.5 - Instale o Docker baixado ```sudo sh get-docker.sh```.

  1.6 - Execute o Docker Swarm com o IP do seu VPS ```docker swarm init --advertise-addr=IP_DO_SERVIDOR```.

  1.7 - Crie duas redes no Docker, uma para o Portainer e outra para o Traefik ```docker network create --driver=overlay container_network``` e ```docker network create --driver=overlay traefik_public```.

  1.8 - Vá até a raiz das pastas do linux (para facilitar achar depois) usando o comando ```cd ..``` até chegar no "/". Ali crie uma pasta para o portainer ```mkdir portainer``` e entre nela ```cd portainer```.

  1.9 - Crie um arquivo .yaml em que será a receita para o portainer usando o seguinte comando: ```nano portainer.yaml``` e cole isso com o botão direito do mouse (por conta do SO ser Linux): 
  ```
  version: "3.8"
  
  services:
    agent:
      image: portainer/agent:latest
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock
        - /var/lib/docker/volumes:/var/lib/docker/volumes
      networks:
        - agent_network
      deploy:
        mode: global
        placement:
          constraints: [ node.platform.os == linux ]
  
    portainer:
      image: portainer/portainer-ce:latest
      command: -H tcp://tasks.agent:9001 --tlsskipverify
      ports:
        - "9000:9000"
      volumes:
        - portainer_data:/data
      networks:
        - container_network
        - traefik_public
      deploy:
        mode: replicated
        replicas: 1
        placement:
          constraints: [ node.role == manager ]
        #labels:
          #- "traefik.enable=true"
          #- "traefik.docker.network=traefik_public"
          #- "traefik.http.routers.portainer.rule=Host(`SEU_SUBDOMINIO.SEU_DOMINIO`)"
          #- "traefik.http.routers.portainer.entrypoints=websecure"
          #- "traefik.http.routers.portainer.priority=1"
          #- "traefik.http.routers.portainer.tls.certresolver=le"
          #- "traefik.http.routers.portainer.service=portainer"
          #- "traefik.http.services.portainer.loadbalancer.server.port=9000"
  
  networks:
    traefik_public:
      external: true
      attachable: true
    container_network:
      external: true
  
  volumes:
    portainer_data:
      external: true
  ```
Depois do colado, aperte Ctrl + X para fechar o arquivo, ele vai pedir se deseja salvar ou não, digite Y de yes para concordar com as mudanças e depois aperte Enter para definir o mesmo nome de arquivo.

Depois vamos ter que modificar os labels que estão comentados nesse arquivo mais pra frente no roadmap como vc pode ver acima nos que estão com "#" na frente.

Finalize agora pedindo ao Docker para implantar essa stack do Portainer no Docker Swarm usando o comando: ```docker stack deploy -c portainer.yaml portainer```.

## Passo 2 - Acessando Portainer e configurando Traefik

  2.1 - Acesse o portainer no seu navegador colocando na URL o IP externo do seu VPS com :9000 no final. Exemplo: ```IP_EXTERNO_VPS:9000``` (por conta de termos definido a porta 9000 naquele arquivo do yaml do portainer)

  2.2 - Assim que acessar vai aparecer uma tela conforme a imagem abaixo, pode deixar o Username como admin se quiser e coloque uma senha de pelo menos 12 caracteres, aparecerá abaixo enquanto digita se ela é forte o suficiente. Depois de criar o Usuário, ele será a forma como vc vai acessar o portainer para administrar os containers nessa VPS. ![image](https://github.com/dpaes/roadmap-portainer/assets/77445296/967eb591-526b-4bbd-a374-f46a0893d3b0)

  2.3 - Agora vamos criar uma outra Stack só que agora dentro do portainer e não através da CLI do Docker (agora vc vai ver a facilidade do portainer). Ache na esquerda do menu lateral a opção "Stacks", terá somente o portainer e como acesso "limited", pois ele não foi criado usando o portainer e sim o Docker somente. Agora clique no botão à direita "+ Add Stack" em azul para criarmos a Stack do Traefik. E cole o código abaixo lembrando de trocar o email aonde está escrito ```SEU_EMAIL@gmail.com``` para o seu email pois é o seu email que aparecerá no certificado SSL quando o traefik gerar o certificado para cada página de aplicação que você definir dentro do portainer em uma outra stack.
  ```
  version: '3.8'
  
  services:
    traefik:
      image: traefik:v2.11
      command:
        - --providers.docker=true
        - --entrypoints.web.address=:80
        - --entrypoints.websecure.address=:443
        - --providers.docker.exposedbydefault=false
        - --providers.docker.swarmMode=true
        #defina a mesma rede que você criou para o traefik
        - --providers.docker.network=traefik_public
        - --providers.docker.endpoint=unix:///var/run/docker.sock
        # Config para SSL Lets Encrypt
        # altere para seu e-mail
        - --certificatesresolvers.le.acme.httpchallenge.entrypoint=web
        - --certificatesresolvers.le.acme.email=SEU_EMAIL@gmail.com
        - --certificatesresolvers.le.acme.storage=/letsencrypt/acme.json
        - --certificatesresolvers.le.acme.tlschallenge=true
        # Global HTTP -> HTTPS
        - --entrypoints.web.http.redirections.entryPoint.to=websecure
        - --entrypoints.web.http.redirections.entryPoint.scheme=https
        #- --api
        #- --log.level=DEBUG
      ports:
        - "80:80"
        #- "8080:8080" # porta do painel do traefik, caso queira ver todas as rotas.
        - "443:443"
      volumes:
        - traefik_certificates:/letsencrypt
        - "/var/run/docker.sock:/var/run/docker.sock:ro"
      deploy:
        mode: replicated
        replicas: 1
        placement:
          constraints:
            - node.role == manager
      networks:
        - traefik_public
  
  volumes:
    traefik_certificates:
      external: true
      name: certificados
  
  networks:
    traefik_public:
      external: true
  ```

  Antes de salvar, desabilite o "Enable access control" e clique depois em Deploy the Stack.

  2.4 - Depois de publicar o Traefik você precisará fazer o apontamento de DNS no seu dominio, dependendo aonde você contratou o seu dominio, seja diretamente no Registro BR ou com uma empresa que permite a compra de dominios também, será necessário você fazer o apontamento, criando um subdominio (TIPO CNAME) a sua escolha (exemplo: portainer) e o valor ser o seu dominio (SEU_DOMINIO.com.br) e no seu dominio (TIPO A) colocar como valor o IP da sua VPS (se você não tivesse feito isso ainda). Assim quando o traefik identificar que vc está tentando acessar o seu site com o subdominio.dominio, ele vai apontar para o IP:PORTA correta.

  2.5 - Depois do apontamento precisará mudar novamente o arquivo portainer.yaml para ele agora funcionar a partir do seu subdominio.dominio que você definiu no apontamento de DNS. Acesse o caminho nas pastas do linux aonde você criou a pasta /portainer usando ```cd portainer``` se vc não estiver já nela no terminal, caso já esteja, apenas digite o comando: ```nano portainer.yaml``` e comente o trecho "ports:" e "- 9000:9000" conforme a imagem abaixo 
  
  ![image](https://github.com/dpaes/roadmap-portainer/assets/77445296/c1c7ce9e-c609-4ed9-afb3-e29ccc45bfea)
  
  E também remova os comentários nos "labels" e pode comentar ou remover a linha em que o traefik utiliza o network "traefik_public" pois não será usado. imagem exemplo abaixo: 
  
  ![image](https://github.com/dpaes/roadmap-portainer/assets/77445296/ed589f99-c8aa-49cd-b7c9-9c117b19b94d)

  Apertar Ctrl + X pra sair do editor, Y pra aceitar a modificação e Enter pra salvar no mesmo arquivo a modificação.

  2.6 - agora fazer um update no portainer com as modificações do arquivo dele, usando o mesmo comando de deploy usado na primeira vez: ```docker stack deploy -c portainer.yaml portainer```.

  Agora só tentar acessar no seu navegador o portainer pelo subdominio.dominio que você definiu e pronto, agora você tem como criar suas aplicações em stacks usando o portainer de forma mais rápida e de fácil manutenção. Colocarei em outros repositórios aplicações open-source que uso e mostrarei como configura-las usando o portainer (que é ridículo de facil).
