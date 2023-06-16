# Como Usar o Traefik para proxy e autenticação de certificado

O arquivo do docker-compose está com todas as configurações de proxy do traefik inseridas nos comandos de inicialização, está configurado para sempre redirecionar requisições na porta **80 (web)** para a porta **443 (websecure)**.

## Arquivo .env

A inicialização do docker-compose exige que exista um **.env** na mesma pasta do arquivo contendo duas informações, **USER_EMAIL** que é o email que será usado para registro no letsencrypt e **DASHBOARD_URL** que é referente a url que o traefik irá expor o dashboard com as informações do serviço.

## Métricas

Em adicional as portas 80 e 443 a porta **8080** também está reservada para expor métricas caso faça uso do Prometheus ou outra ferramenta de captura de métricas. As métricas poderão ser acessadas da forma que é mostrada no exemplo `http://exemplo.com:8080/metrics`

## Dashboard

O uso do dashboard é totalmente opcional, caso n queira fazer uso do mesmo basta mudar para **false** na parte de comandos de inicialização de contêiner  `--api` e **comentar** todas estas linhas no Label do serviço do traefik no docker-compose:

> "traefik.http.routers.dashboard.rule=Host(`${DASHBOARD_URL}`)"
> "traefik.http.routers.dashboard.tls=true"
> "traefik.http.routers.dashboard.tls.certresolver=letsencrypt"
> "traefik.http.routers.dashboard.service=api@internal"

## Fazendo uso do traefik para proxy e certificação

Antes de inicializar o arquivo do docker-compose do traefik é necessário que seja criada uma rede no docker para que o traefik faça uso. No caso das configurações deste docker-compose a rede tem que se chamar **traefik**. Para criar uma rede com o docker basta executar o comando `docker network create traefik`.

Agora após a inicialização do contêiner via docker-compose o traefik passara a usar está rede associada as portas 80 e 443 para receber qualquer requisição que chegar ao servidor e redirecionar para o contêiner com o serviço adequado, usando como forma de identificação o **hostname** para o qual a requisição é destinada.

Mas para fazer essa identificação é necessário que todos os demais contêineres que você vão rodar no servidor e queira fazer uso do serviço de proxy sigam alguns padrões simples. Todo contêiner que for fazer uso do traefik deve conter **Labels** que indiquem uma configuração de rotas que vai ser explicada mais abaixo e o contêiner tem que estar dentro da **rede traefik** que foi criada anteriormente, não exclusivamente ele também pode ter uma rede interna para conversar com os demais serviços que não façam uso do proxy ou para troca interna de informações entre serviços que trabalham em conjunto.

## Labels

As Labels são a forma que o traefik usa para endereçar e saber para onde redirecionar cada requisição que chega para ele através do endereçamento por **hostname**. Alem disso as labels servem para alguns outros propósitos como indicar qual vai ser a unidade certificadora do contêiner, por qual porta o serviço vai receber as entradas de comunicação, uso de middlewares configurados. Para o nosso uso de de proxy e certificação vai ser necessário o uso das seguintes labels:

>labels:
>>traefik.enable: "true"
>>traefik.http.routers.example.rule: "Host(`${HOSTNAME}`)"
>>traefik.http.routers.example.entrypoints: "websecure"
>>traefik.http.routers.example.tls: "true"
>>traefik.http.routers.example.tls.certresolver: "letsencrypt"

Para fins de explicação o nome da rota criada é **example** mas no caso cada projeto que vá rodar em um contêiner que fará uso do proxy tem que ter um nome de rota único. Explicando brevemente cada label, `traefik.enable: "true"` indica que o contêiner usará o traefik como aplicação de proxy todos os demais que não farão uso devem conter essa label com o termo `false`.  

A label `traefik.http.routers.example.rule: "Host()"` é a label principal que vai conter os hostnames que esse contêiner responderá.

A label `traefik.http.routers.example.entrypoints: "websecure"` indica por qual entrypoint cadastrado no traefik este contêiner deve receber a requisição, no caso 'websecure' indicia que todas as requisições chegaram para ele pela porta 443.

A label `traefik.http.routers.example.tls: "true"` indica se a requisição deve vir via protocolo TLS ou não.

A label `traefik.http.routers.example.tls.certresolver: "letsencrypt"` indica qual deve ser a unidade certificadora que fará a validação do certificado TLS caso tenha sido usado o protocolo. no nosso caso como pode ser visto na configuração do traefik o **letsencrypt** é a unidade certificadora que validará o certificado, mas caso você possua outra empresa privada que faça essa validação basta inserir nas configurações do traefik e dar um nome único ao certresolver.

## Tudo pronto

Com o traefik rodando e com as configurações acima feitas no nos contêineres basta inicializar eles que eles devem passar a responder as requisições que chegarem ao servidor com o hostname cadastrado para eles. Inicialmente o traefik fará uso de um certificado genérico para rodar o contêiner e você receberá a famosa mensagem do Google que o certificado n é confiável. Para resolver isso e validar o certificado basta ter certeza que o hostname em questão estará exposto a internet e reiniciar o contêiner do traefik, que ai ele fará a comunicação com os servidores do letsencrypt e efetuara a autenticação dos certificados válidos para o endereço.
