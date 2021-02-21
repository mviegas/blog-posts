---
title: "RabbitMQ e AMQP: conceitos básicos sobre o protocolo de transporte"
description: ""
date: 2999-03-08
draft: true
language: pt
tags: ["dotnet", "rabbitmq", "distributedsystems", "microservices"]
---

# *tl/dr*

Arquiteturas distribuídas trazem uma série de vantagens como desacoplamento e assincronicidade. Contudo, disponibilidade de infra-estrutura e confiabilidade de que dados não serão perdidos ao saírem de uma dos nós dessa arquitetura se tornam requisitos que merecem a máxima atenção. Neste post, falarei um pouco mais sobre o protocolo AMQP e a estrutura que o RabbitMQ utiliza em cima dele para permitir configurações confiáveis na distribuição de mensagens.

---

## Conceitos básicos do AMQP

**AMQP**, ou *Advanced Message Queue Protocol*, um protocolo de transporte de mensagens criado à partir de uma especificação que dita como aplicações clientes e *brokers* de mensageria se conectam e comunicam, ou seja, como eles trocam mensagens. Como dito no [post anterior](https://dev.to/mviegas/pt-br-introducao-ao-rabbitmq-com-net-core-15oc), ele é apenas um dos protocolos suportados pelo RabbitMQ, o nosso broker.

O papel de um *broker* no AMQP é o seguinte:
- Receber a mensagem de um ou mais **produtor(es)** (*publisher/producer*).
- Rotear essa mensagem para um ou mais **consumidor(es)** (*consumer*).

A *especificação* do AMQP é composta por **comandos** descritos através de **classes** e **métodos**, o que facilita a abstração do protocolo na construção de bibliotecas-cliente como a [RabbitMQ.Client](https://github.com/rabbitmq/rabbitmq-dotnet-client) para .NET, uma vez que um paralelo à classes e métodos da orientação à objetos pode ser feito. 

- As *classes* definem um escopo de funcionalidades
- Os *métodos* definem ações e tarefas que serão executadas
- Uma chamada à um método de uma classe é também descrita como um *comando*

Antes de entender que comandos são esses, é preciso entender como os dados são tranportados no *AMQP* através de **conexões**, **canais** e **frames**.

---

## Conexões e canais

O RabbitMQ utiliza o padrão *RPC (remote procedure call)* para realizar a maior parte da sua comunicação. Em resumo, *RPC* é um pattern que permite que um computador (*client*) execute um programa/ação em outro (*server*). Ou ainda, de maneira semelhante à uma web API: um cliente faz um *request* e recebe um *response* do servidor e isso pode ser feito de maneira *síncrona* ou *assíncrona*. Há muito mais o que se falar sobre o RPC, mas pro que precisamos isso basta. Se você quiser saber mais, [esse livro](https://www.amazon.com/Distributed-Systems-Concepts-Design-5th/dp/0132143011) sobre sistemas distribuídos é uma excelente fonte. Dito isso, vamos ver como o RabbitMQ usa o RPC para criar uma conexão entre um cliente e o broker. 

Um *client* inicia uma **conexão** com um *broker* a partir de 3 chamadas RPC realizadas através do AMQP:
- O *client* envia uma saudação para o *broker* através de um `protocol header`. É algo como um *"E aí, beleza? To por aqui!"*
- O *broker*, então, envia o comando `Connection.Start` para o client
- Por fim, o *client* responde enviando um comando `Connection.StartOk`

Quando essa sequência de requests foi executada com sucesso, uma conexão entre *client* e *broker* é  estabelecida, e a partir daí o *client* pode enviar uma série de outros *comandos* para o *broker* através de **canais**.

Canais são o meio especificado no AMQP para a comunicação de *clients* com um *broker*. Funcionam de forma semelhante à comunicação de dois pontos entre rádios (como walkie-talkies). De forma semelhante à uma frequência de rádio utilizada para conectar as duas pontas, os *canais* utilizam a *conexão AMQP* estabelecida para transmitir informação de uma ponta à outra. É importante dizer que **canais são isolados**, de forma que cada troca/transmissão de mensagens é independente e exclusiva daquele canal no qual ela acontece. Uma única conexão pode ter *múltiplos canais* (para os curiosos, isso é feito através da implementação de um [multiplexador](https://en.wikipedia.org/wiki/Multiplexing)).

---

## Frames

Um **frame** é uma *estrutura de dados* utilizada pelo AMQP para enviar/receber dados entre clientes e brokers. Ele é a unidade básica do protocolo. No proceso de negociação de uma conexão que descrevemos acima, o comando `Connection.Start` é enviado serializado em um frame para o cliente. Nesse comando, `Connection` é a classe e `Start` é o método. Um frame contém todos os argumentos necessários para a execução de um comando. Sua estrutura é a seguinte:

![](https://www.brianstorti.com/assets/images/frame.png)

- **Frame header**: define o *tipo* do frame, *o número do canal* (falaremos mais sobre isso à frente) para o qual o frame está sendo enviado e seu *tamanho* em bytes.
- **Payload**: o corpo da mensagem.
- **End-byte marker**: um byte (o caracter ASCII 206) utilizado para delimitar o final do frame.

### Tipos de frame no AMQP

A especificação do AMQP define cinco tipos de frame:

- **Protocol header frame**: esse frame é utilizado uma única vez, no *greeting* do processo de conexão do *client* ao *broker*, conforme descrito acima
- **Method frame**: é o frame utilizado para levar o request/response RPC do/para o *broker*.
- **Content header frame**: frame que contém o tamanho e as propriedades da mensagem.
- **Body frame**: frame que contém o conteúdo da mensagem.
- **Heartbeat frame**: frame enviado do/para o *broker* para checar que uma *conexão* está disponível e funcionando corretamente. Este frame em especial é um exemplo de como o AMQP é um protocolo RPC, ou seja, com uma comunicação *bidirecional*. Se o RabbitMQ envia um heartbeat para um client e não obtém resposata, a conexão com esse client é fechada.

---

## Exemplo: abstraindo conexões, canais e frames

Como vimos no [post anterior](), o mínimo que preciamos para publicar uma mensagem no broker é que existam uma *exchange* e uma *fila* e que elas estejam associadas (*bind*).

O exemplo abaixo (em C#) cria uma conexão, um canal dentro dessa conexão e envia através desse canal uma mensagem de uma aplicação cliente para o broker. Vamos entender cada passo desses à nível de protocolo de transporte.

> Relembrando: quando uma mensagem é enviada com o nome da *exchange* vazio no parâmetro, ela é na verdade enviada para uma *exchange* default e roteada para uma fila com exatamente o mesmo nome da *routingKey*.

```csharp
using (var connection = connectionFactory.CreateConnection())
using (var channel = connection.CreateModel())
{
    Console.WriteLine("Type your message");

    string inputMessage = Console.ReadLine();

    channel.QueueDeclare(
        queue: "test-queue",
        durable: false,
        exclusive: false,
        autoDelete: false,
        arguments: null);

    string message =
        $"{DateTime.Now.ToString("dd/MM/yyyy HH:mm:ss")} - " +
        $"Message content: {inputMessage}";
		
    byte[] messageBody = Encoding.UTF8.GetBytes(message);

	channel.BasicPublish(
		exchange: string.Empty,
        routingKey: "test-queue",
        basicProperties: null,
        body: messageBody);
}
```

> Disclaimer: o código acima é meramente de exemplo e não deve ser replicado em produção. Existem boas práticas a serem executadas na criação de [conexões](https://www.rabbitmq.com/connections.html) e [canais](https://www.rabbitmq.com/channels.html).

### Criando uma conexão

Aqui temos outro bom exemplo de como a comunicação client-broker é feita através de RPC. Logo na primeira linha, uma conexão é criada com a chamada:

```csharp
var connection = connectionFactory.CreateConnection()
```

Isso é feito da seguinte forma:
- Um *protocol header frame* é enviado do cliente para o broker
    - Client >------- *Greeting* --------> Broker
- Quando o broker recebe esse frame, ele envia um *method frame* com o comando `Connection.Start` para o client
    - Client <---- *Connection.Start* ----< Broker
- O client então responde com outro *method frame* com o comando `Connection.StartOk`
    - Client >---- *Connection.StartOk* ----> Broker

### Criando um canal
```csharp
var channel = connection.CreateModel()
```

### Declarando uma fila
```csharp
    channel.QueueDeclare(
        queue: "test-queue",
        durable: false,
        exclusive: false,
        autoDelete: false,
        arguments: null);
```

A declaração da fila em um canal é feita da seguinte forma:
- Um *method frame* é enviado com o comando `Queue.Declare`. Esse frame contém os parâmetros que definem o nome da fila, se ela é durável, exclusiva, autoDelete ou outros argumentos adicionais.
     - Client >---- *Queue.Declare* ----> Broker
- Outro *method frame* é retornado com o comando `Queue.DeclareOk`
    - Client <---- *Queue.DeclareOk* ----< Broker


### Enviando uma mensagem
```csharp
	channel.BasicPublish(
		exchange: string.Empty,
        routingKey: "test-queue",
        basicProperties: null,
        body: messageBody);
```

O envio de uma mensagem para o RabbitMQ através de um canal é feito da seguinte forma:

- Um *method frame* é enviado com o comando `Basic.Publish` e os parâmetros `exchange` e `routingKey`
- Após esse, é enviado um *content header frame* que contém um comando com as propriedades da mensagem - o argumento `basicProperties`. Há muito o que se falar sobre elas, mas por hora basta saber que é ali que especificamos algumas coisas para se criar um *contrato* da mensagem
- Por fim, o body da mensagem é enviado um ou mais *body frame(s)* a depender de seu tamanho. Como exemplo, [veja esse link](https://github.com/rabbitmq/rabbitmq-dotnet-client/blob/d0c5123d3d402b66d5551f8a541c1aad961db845/projects/client/RabbitMQ.Client/src/client/impl/Command.cs) que contém a implementação do envio de *comandos* através de *frames* da lib .NET [RabbitMQ.Client](https://github.com/rabbitmq/rabbitmq-dotnet-client/).

---

## Lidando com erros

A especificação do AMQP lida com erros da maneira mais 

---

## Resumo

Normalmente usaremos uma biblioteca já implementada e consolidade pra desenvolver aplicações que se comunicam com brokers RabbitMQ. Entender, contudo, um pouco mais do protocolo AMQP que elas implementam à nível de transporte pode ser bem útil para entender comportamentos estranhos que possam acontecer por falta de conhecimento na configuração e porque certas escolhas ocasionam na renúncia de outras (*tradeoffs*). Conexões e canais são a base da comunicação via AMQP.


