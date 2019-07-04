# Tutorial iniciante em mqtt 

### O que é MQTT?

O MQTT (Message Queuing Telemetry Transport) é um protocolo de mensagem com suporte para a comunicação assíncrona entre as partes. Um protocolo de sistema de mensagens assíncrono desacopla emissor e receptor da mensagem tanto no espaço quanto no tempo. Como descrito no [site oficial](http://mqtt.org/), o MQTT é o protocolo perfeito para IOT  por ter sido projetado para minimizar o uso de largura de banda da rede e poupar os recursos dos dispositivos, ao mesmo tempo em que tenta garantir a confiabilidade e um certo grau de garantia de entrega. Apesar de seu nome, ele não tem nada a ver com filas de mensagens, na verdade, ele usa um modelo publisher/subscriber.  

### Entendendo publish subscriber…

A arquitetura básica do publish/subscriber é formada pelos seguintes conceitos representados na imagem abaixo:

![pasted image 0](https://user-images.githubusercontent.com/20146828/60650150-eed7c580-9e19-11e9-8bb1-5789148bbcc8.png)

**Publisher**: O dispositivo que publica mensagens em um determinado tópico.  
**Subscriber**: The device which receives messages by subscribing to a particular topic. O dispositivo que recebe mensagens por assinar um determinado tópico.  
**Broker**: Uma entidade central, o Broker, que recebe mensagens dos publishers e as reencaminha para subscribers cadastrados em determinado tópico.  

Uma analogia semelhante seria a assinatura de um RSS em uma página de notícias, por parte do leitor.

### Os motivos pelos quais MQTT é a melhor opção em IoT…

Um dos pontos fortes desse protocolo é o desacoplamento entre o publish e o subscribe. Os dois elementos têm conhecimento zero um do outro, permitindo uma independência muito grande entre os elementos que populam as mensagens no broker e os elementos que irão consumir essa informação.  

E essa independência favorece, e muito, a adoção do protocolo no ambiente móvel e, principalmente, no mundo da Internet das Coisas. Justamente porque a heterogeneidade dos elementos, tanto no publish quanto no subscribe, é muito grande. Imaginem a quantidade de plataformas, linguagens de programação, protocolos de comunicação e tecnologias para um software poder atingir um número massivo de clientes em soluções IoT . Um cliente pode ser um sensor de IoT em campo ou um aplicativo em uma nuvem que processa dados de IoT.  

Como as mensagens do MQTT são organizadas por tópicos, o desenvolvedor tem a flexibilidade de especificar que determinados clientes somente podem interagir com determinadas mensagens, como podemos ver na imagem a seguir:

![pasted image 1](https://user-images.githubusercontent.com/20146828/60654061-9d333900-9e21-11e9-9b9e-6b64f3f01cac.jpg)

### Vamos ver como funciona...

Usaremos o [cloudmqtt](https://www.cloudmqtt.com/) como broker na nuvem por ser extremamente acessível e disponibilizar uma interface websocket, outras opções mais empresarias são o flespi mqtt , o hivemq e o myQttHub. 

E instalaremos o [mosquitto](https://mosquitto.org/), um broker opensource, que inclui o mosquitto-pub e o mosquitto-sub, clientes simples (via CLI) que permitem testar a comunicação do protocolo. Como o objetivo do tutorial é enteder o protocolo, vamos manter a simplicidade e usar o básico, mas existem diversos outros brokers que vc pode consultar nessa [wiki](https://github.com/mqtt/mqtt.github.io/wiki/brokers), e claro, diversas ferramentas para simular [clientes mqtt](https://github.com/mqtt/mqtt.github.io/wiki/tools) ou [bibliotecas](https://github.com/mqtt/mqtt.github.io/wiki/libraries) para diversas linguagens. 

Instalando o mosquitto:
```
sudo apt-get update -y && sudo apt-get upgrade -y
```
```
sudo apt-get install mosquitto
```
```
sudo apt-get install mosquitto-clients
```
Concluída a instalação, vamos testar os clients pub e sub em localhost. Vamos abrir dois terminais, um para o sub e outro para o pub.

No primeira janela, vamos executar o sub:
```
mosquitto_sub -t mytopic
```

No segunda execute o pub:
```
mosquitto_pub -m mymessage -t mytopic
```
Como você acabou de ver, o subscriber recebeu a mensagem “mymessage” que o publisher disparou, já que estava assinando ao tópico “mytopic”. Abra outros terminais e inscreva mais subs no mesmo tópico e depois faça mais uma publicação com o pub.

Agora vamos reexecutar os comandos anteriores acrescentando o parametro -d no final para visualizar o debug. Na documentação você pode verificar os parametros para os comandos [mosquitto_sub](https://mosquitto.org/man/mosquitto_sub-1.html) e [mosquito_pub](https://mosquitto.org/man/mosquitto_pub-1.html).

Execute o sub:
```
mosquitto_sub -t mytopic -d
```
E depois o pub:
```
mosquitto_pub -m mymessage -t mytopic -d
```
Agora com o debug você pode ver as mensagens trocadas na comunicação do pub e sub com o broker, tais como  CONNECT, CONNACK, SUBSCRIBE, SUBACK. Então vamos entender melhor cada uma delas e como trabalha o protocolo MQTT.  

### O funcionamento do protocolo...

>Primeiro, o cliente conecta-se ao broker enviando uma mensagem CONNECT. 

A mensagem CONNECT pede para estabelecer uma conexão do cliente com o broker. A mensagem CONNECT tem os seguintes parâmetros de conteúdo: cleanSession, username, password, lasWillTopic, lastWillQos, lasWillMessage, keepAlive. 

Vamos destacar o cleanSession, esse flag bit especifica  se a conexão é persistente ou não. O client e o broker podem armazenar o estado da sessão para permitir troca confiável de mensagens em uma sequência de conexão. Os estados de sessões consistem em QoS.

O QoS (quality of service) define três tipos/levels:  
* QoS - 0: Não garante que a mensagem tenha sido entregue com sucesso.  
* QoS - 1: Garante que a mensagem será entregue pelo menos uma vez. Mas pode ser enviada mais de uma vez.  
* QoS - 2: Garante que a mensagem seja enviada apenas uma vez.  

Em uma conexão persistente quando houver perda de conexão, as assinaturas e as mensagens possivelmente perdidas ( not acknowledged) serão guardadas.

Os parâmetros que se iniciam com lastWill são flag bits que oferecem um serviço de “último desejo” em que uma mensagem desejada será publicada quando uma conexão for encerrada inesperadamente, como por exemplo, se o cliente não der sinal de vida dentro do limite do parâmetro keepAlive, que especifica o intervalo de tempo em que o client precisa efetuar ping no broker para manter a conexão ativa.

>Veja que após o mosquitto_sub e o mosquitto_pub enviarem o CONNECT, ambos receberam o CONNACK como resposta. 

Os parâmetros do CONNACK são: sessionPresent que indica se a conexão já tem uma sessão persistente, ou seja, se a conexão já tem tópicos assinados e receberá a entrega de mensagens ausentes. returnCode que retorna 0 em caso de sucesso ou outros valores que indicam a causa da falha no caso de erro. Se o broker envia um valor diferente de zero, a conexão é fechada.

>Com a conexão estabelecida, o mosquitto_sub enviou a mensagem SUBSCRIBE e o mosquitto_pub enviou a mensagem PUBLISH.

A mensagem PUBLISH enviada possui um tópico e um carga útil de dados, tais dados que quando recebidos pelo Broker serão encaminhados para todos os assinantes do respectivo tópico. Os parâmetros são: topicName, qos, dup, retain, payLoad. Em topicName está a string com o nome do tópico no qual a mensagem em payLoad será publicada. O parâmetro QoS como dito anteriormente define o nível de qualidade do serviço de entrega da mensagem. Dup indica se é a primeira tentativa de envio do pacote ou não.  Já a flag retain vai sinalizar se o broker deve reter a mensagem como a última mensagem conhecida do tópico ou se vai descartar e manter a mensagem que já possui com ele.

O retorno para o PUBLISH é um pacote PUBACK ou um pacote PUBREC, sendo o primeiro uma resposta para um pacote QoS 1 e o segundo uma resposta para um pacote QoS 2.

>Como visto no terminal o mosquitto_pub após fazer sua publicação envia a mensagem DISCONNECT e encerra sua conexão.

A mensagem SUBSCRIBE enviada ao broker pelo client carrega as assinaturas em um ou mais tópicos, tais assinaturas que vão permitir receber as publicações desses tópicos. Vale ressaltar que mensagens PUBLISH são enviadas de clients para o broker e do broker para os clients como forma de enviar dados da aplicação, enquanto a mensagem SUBSCRIBE é somente para o client comunicar ao broker os seus tópicos de interesse.

Os parâmetros de SUBSCRIBE são qos e topic. Um pacote cujo payload não tiver pelo menos um tópico e um qos está violando o protocolo.

Quando o broker recebe um pacote SUBSCRIBE do client a resposta será um pacote SUBACK com o mesmo identificador de pacote do pacote que está reconhecendo e os devidos returncodes que indicam o sucesso ou a falha da assinatura no tópico. Enquanto o UNSUBSCRIBE permite cancelar a assinatura do tópico e tem como resposta um UNSUBACK

>Como visto no terminal, o mosquitto_sub recebe as publicações e fica pingando na espera de novas publicações.

### Observando os pacotes... 
Agora vamos nos conectar ao cloudmqtt (para que você possa acompanhar os pacotes do MQTT, use o wireshark), um broker na nuvem com client websocket. Para criar uma instância gratis é necessário apenas se logar e com poucos cliques sua instância é provisionada. Após criada a instância, você usará as informações da aba “details” para se conectar ao broker.

![Captura de tela de 2019-07-03 09-38-14](https://user-images.githubusercontent.com/20146828/60663927-f6599780-9e36-11e9-8895-f22115e8e107.png)

Execute o comando a seguir com os dados da sua instancia, o parametro -h se refere ao server, o -p se refere a porta, o -t se refere ao tópico, o -u é o user e o -P a senha.
```
mosquitto_sub -h postman.cloudmqtt.com -p 13244 -t meutopico -u zvdqauay -P Nl0wujLOxrKD
```
Agora vá na aba websocket ui e envie uma mensagem ao tópico que vc criou e veja ela chegando até o seu client mosquitto_sub. Acompanhe os pacotes de controle no wireshark.

### Continue seus estudos em MQTT...  
Depois de dominar os conceitos de MQTT, dê uma olhada no bevywise (que permite simular sua solução IoT com quantos dispositivos vc quiser) e também em Node + mqtt, esse tutorial [aqui](https://blog.risingstack.com/getting-started-with-nodejs-and-mqtt/) mostra que existem muitas possibilidades. Veja um exemplo em Node abaixo, simples não é mesmo?! A biblioteca [mqqtjs](https://github.com/mqttjs/MQTT.js) é bem documentada.

```js
var mqtt = require('mqtt');
var time = require('time');
 
var now = new time.Date();
 
var broker = 'localhost';
var port = 1883;
var message = now.toString() + ' - This is the message'
var topic = 'test/mqtt'
 
var client = mqtt.createClient(port, broker);
 
client.publish(topic, message);
client.end();
```

