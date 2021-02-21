---
title: "RabbitMQ e AMQP: desvendando os acknowledgements"
description: ""
date: 2020-03-20
draft: true
language: pt
tags: ["dotnet", "rabbitmq", "distributedsystems", "microservices"]
---

# *tl/dr*

O objetivo desse post é entender um pouco mais sobre a confirmação de mensagens processadas por um *consumidor*, como essa confirmação (ou rejeição) acontece e sobre estratégias para lidar com mensagens rejeitadas.

## Sistemas distribuídos e suas complicações

Uma dificuldade nesse tipo de arquitetura é garantir que uma mensagem disparada por um lado seja processada com sucesso por quem a recebe. Essa é uma das motivações de o *RabbitMQ* ter uma implementação dedicada à essa tarefa: a [API de *acknowledgements*](https://www.rabbitmq.com/confirms.html). Essa API garante que o *broker* saiba se uma mensagem foi entregue e aceita ou rejeitada por um determinado *consumidor*, o que levanta uma série de possibilidades como, por exemplo:

- resiliência em variações na topologia da rede
- informação de sucesso/erros no processamento

---

## Acknowledgements (ou simplesmente *acks*)

Um *acknowledgement* nada mais é que uma confirmação emitida por um *consumidor* de uma mensagem para o *message broker* de que essa mensagem foi recebida e processada com sucesso. Toda vez que um *ack* é emitido, o RabbitMQ entende que a mensagem foi processada e que ela pode ser descartada.

> Um *ack* não é algo específico do RabbitMQ, mas sim um método do protocolo AMQP. VOcê pode ver mais sobre isso [nesse link](https://www.rabbitmq.com/amqp-0-9-1-reference.html#class.channel).

Existem duas formas de se emitir um *ack*:

- Automática: a confirmação é feita no momento em que a mensagem sai do broker para o consumidor. Esse é o significado literal de **fire and forget** e esse tipo de prática é altamente **não** recomendada. Imagine o seguinte cenário: a mensagem sai do broker para o consumidor mas no trajeto, logo antes de chegar, uma variação na rede derruba a conexão/o canal por onde a mensagem foi enviada. Como o *ack* já foi emitido no disparo da mensagem, o broker já retirou ela da fila e ela é, portanto, perdida. 

#### Exemplo de um *ack* automático: 

```csharp
var consumer = new EventingBasicConsumer(channel);

consumer.Received += (sender, eventArgs) =>
{
    var message = Encoding.UTF8.GetString(eventArgs.Body);
    Console.WriteLine(Environment.NewLine + "[New message received] " + message);
};

channel.BasicConsume(queue: "tests", autoAck: true, consumer: consumer);
```

- Manual: aonde o consumidor deve confirmar manualmente o ack após processar a mensagem. Os *acks* manuais reduzem um pouco a vazão (throughput) de mensagens, mas permitem aos consumidores um maior controle das mensagens que chegam. Isso pode ser interessante, pois existe uma predefinição do canal pelo qual a mensagem é enviada que limita o número de mensagens entregues, garantindo assim que o consumidor não fique sobrecarregado.

#### Exemplo de um *ack* manual: 

```csharp
var consumer = new EventingBasicConsumer(channel);

consumer.Received += (sender, eventArgs) =>
{
    var message = Encoding.UTF8.GetString(eventArgs.Body);
    Console.WriteLine(Environment.NewLine + "[New message received] " + message);
    channel.BasicAck(deliveryTag: eventArgs.DeliveryTag, multiple: false);
};

channel.BasicConsume(queue: "tests", autoAck: false, consumer: consumer);
```
A chamada do método `BasicAck` é a responsável por fazer o *ack* manual. Este método tem os seguintes parâmetros:

- `delieveryTag`: *delivery tags* são identificadores que cada entrega de mensagem possui. É através delas que é identificado que o *ack* foi feito para uma determinada mensagem.
- `multiple`: este parâmetro é utilizado para realizar um *ack* em lote. **TODO TODO TODO TODO**

> É importante frisar que quem detém o conhecimento se a mensagem foi ou não processada com sucesso é o *consumidor*. Portanto, cabe somente à ele decidir e informar no momento em que se registra no broker, o tipo de *ack* que ele emitirá: automático ou manual.

---

## O outro lado da moeda: os *nacks*

Um *nack* nada mais é que um *ack* ao contrário: é o consumidor dizendo que aconteceu algum erro no processamento da mensagem. Ele pode ser emitido da seguinte forma:

### Exemplo: 

```csharp
```


---

## Estratégia para lidar com mensagens rejeitadas/expiradas: dead-letters

Uma *dead-letter*, comumente também chamada de *dlx*, nada mais é que uma *exchange* configurada para receber mensagens que:
- foram rejeitadas pelo consumidor;
- tiveram o tempo de vida na fila expirado;
- uma fila rejeitou pois estourou o número máximo de mensagens que ela permite.

É possível se configurar uma dead-letter através de uma *policy* via CLI do RabbitMQ ou se passando um *argumento na declaração de uma fila*.

> Não mostrarei nenhum exemplos de configuração de *dead-letters* com *policy* nesse post, mas você pode ler mais sobre [nesse link](https://www.rabbitmq.com/parameters.html#policies).

Para setarmos uma exchange de dead-letter numa fila, basta declararmos a fila com o seguinte argumento: `x-dead-letter-exchange`.

```csharp
channel.ExchangeDeclare("my-dead-letter", "direct");

var args = new Dictionary<string, object>();
args.Add("x-dead-letter-exchange", "my-dead-letter");

channel.QueueDeclare(
    queue: "my-dead-letter",
    durable: false,
    exclusive: false,
    autoDelete: false,
    arguments: args);
```

# Resumo
