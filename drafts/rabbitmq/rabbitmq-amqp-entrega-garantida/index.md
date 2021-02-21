---
title: "RabbitMQ e AMQP: garantindo a entrega e o processamento de mensagens"
published: "false"
language: pt
date: "2020-03-31"
description: 
tags: ["dotnet", "rabbitmq", "distributedsystems", "microservices"]
---

# *tl/dr*

Este post tem como objetivo mostrar duas features do RabbitMQ implementadas à partir da especificação do AMQP que tem como objetivo garantir a entrega de mensagens de *publisher* para o *broker* e o processamento da mensagem entregue pelo *broker* para um *consumer*, exemplos aonde isso pode ser útil e tradeoffs da adoção dessas funcionalidades.

---

## Garantindo o roteamento da mensagem com a flag *mandatory*

De acordo com a [especificação do AMQP](https://www.rabbitmq.com/amqp-0-9-1-reference.html), o comando `Basic.Publish` possui uma flag chamada `mandatory`. Essa flag é responsável por garantir que a mensagem não foi somente entregue para o *message broker*, mas devidamente *roteada* de acordo com os parâmetros definidos no comando. Caso a mensagem seja publicada com `mandatory=false`, o *broker* vai se desfazer da mensagem não-roteada sem emitir nenhum aviso.

Uma vez que o comando `mandatory=true` e a mensagem publicada não tenha sido roteada, o *broker* envia para o *publisher* o comando RPC `Basic.Return`. Na biblioteca [RabbitMQ.Client]() **TODO : COLOCAR LINK AQUI** isso é feito através da implementação de um evento `BasicReturn` do mesmo *canal* por onde a mensagem foi enviada. A implementação em C# é feita através de evento, pois o comando AMQP `Basic.Return` é executando assincronamente e pode ocorrer a qualquer hora após a mensagem ser publicada. Deste modo, a execução do programa não fica travada, uma vez que o tratamento da mensagem não roteada é feito através da assinatura de um handler para um evento.

> O comando `Basic.Return` é executado através de um *method frame*, junto com um *content header frame* e um *body frame* de maneira semelhante ao comando `Basic.Publish`. Só que no caso do *Return*, o comando é executado pelo *broker*, enquanto no caso do *Publish*, o comando é executado pelo *publisher*.

### *Exemplo*

Abaixo criamos o comando `Basic.Publish` com a flag `mandatory=true`. Nesse caso, toda vez que uma mensagem for enviada e não for roteada, o comando `Basic.Return` será executado pelo broker e a nossa aplicação produtora de mensagens executará o evento assinado para o *canal* através do qual a mensagem foi enviada. Caso o roteamento seja feito com sucesso, nenhuma mensagem será exibida.

> *Disclaimer*: os códigos aqui mostrados são **meramente exemplos** sobre a implementação do AMQP em .NET. Não copie e cole este código em produção! Você pode encontrar este exemplo completo para execução [neste gist](https://gist.github.com/mviegas/71edf43a658070a19aefda04172676fa).

```csharp
    using (IModel channel = connection.CreateModel())
    {
        channel.BasicReturn += (sender, eventArgs) =>
        {
            string message = System.Text.Encoding.UTF8.GetString(eventArgs.Body);

            Console.WriteLine($"Message [{message}] rejected because it could not be routed to exchange [{eventArgs.Exchange}] with routing key [{eventArgs.RoutingKey}]");
        };

        byte[] messageBody = System.Text.Encoding.UTF8.GetBytes("Test message");

        channel.BasicPublish(
            exchange: string.Empty,
            routingKey: routingKey,
            mandatory: true,
            basicProperties: null,
            body: messageBody);
    }
```

### Resumindo a flag *mandatory*

Utilize esta flag quando quiser ter certeza de que a mensagem foi roteada com sucesso pela exchange para alguma fila associada.

---

## Garantindo a entrega e o processamento da mensagem com *publisher confirms*

Uma vez que a comunicação *publisher <-> broker* é feita através de *rede*, uma premissa fundamental é a seguinte: **falhas na rede vão acontecer, portanto esteja preparado para elas**. Com base nisso, o RabbitMQ implementou a feature do *publisher confirms*, que não consta na especificação do AMQP, mas foi implementada como uma extensão do protocolo de maneira à adicionar *resiliência* à publicação de mensagens. 

Quando a feature *publisher confirms* é habilitada, é criado um *contrato* entre o *publisher* e o *broker* através do qual o *publisher* será sempre notificado se uma mensagem foi recebida ou não pelo *broker*. Isso é feito através da execução de uma chamada RPC com os comandos AMQP `Confirm.Select` e `Confirm.SelectOk`. 

De acordo com a [especificação do protocolo](https://www.rabbitmq.com/amqp-0-9-1-reference.html#class.confirm), o comando `Confirm.Select` é enviado do *publisher* para o *broker* como uma solicitação de que ele seja notificado através de *acks* sobre a entrega de uma determinada mensagem. O *broker* então, deve responder com um comando `Confirm.SelectOk` para que o *publisher* tenha a confirmação devidamente habilitada. Uma vez habilitada, o *publisher* envia uma mensagem para o *broker* através do comando `Basic.Publish` e o *broker*, por sua vez, responde com um comando `Basic.Ack` ou `Basic.Nack`, ambos contendo a `delivery tag` que identifica a mensagem enviada. 

Ainda de acordo com a [documentação oficial](https://rabbitmq.docs.pivotal.io/35/rabbit-web-docs/confirms.html), o comando `Basic.Ack` será **sempre** enviado pelo *broker* para confirmar o recebimento da seguinte maneira, independentemente se a mensagem foi roteada com sucesso ou não. Esse *ack* acontece da seguinte forma:

- Para mensagens não-roteadas:
    - O *ack* será enviado assim que a *exchange* verificar que a mensagem não pode ser entregue à nenhuma fila;
    - Se a mensagem também for enviada com a flag `mandatory=true`, o comando `Basic.Return` é enviado antes do *ack*,
- Para mensagens roteadas:
    - O *ack* é enviado quando a mensagem foi recebida por todas as filas. No caso de mensagens *persistentes*, isso acontece quando a mensagem foi escrita no disco,

O comando `Basic.Nack` só será executado se ocorrer alguma falha no processo Erlang do broker que impeça da mensagem chegar na *exchange* e ter o roteamento verificado.

### *Exemplo*

> Vale o mesmo *disclaimer* acima: não copie e cole este código em produção. Ele é um exemplo simples de como habilitar a feature. O código completo do exemplo abaixo está [neste gist](https://gist.github.com/mviegas/8adff03f69cf1c3beabd40b2850b0c31).

```csharp
using IModel channel = connection.CreateModel();
{
    // Emite o comando Confirm.Select para o broker através do canal
    channel.ConfirmSelect();

    // Inscrição por eventos para receber assincronamente os comandos Basic: Ack, Nack e Return
    channel.BasicAcks += (sender, eventArgs) => Console.WriteLine($"Message [{eventArgs.DeliveryTag}] was acked!");
    channel.BasicNacks += (sender, eventArgs) => Console.WriteLine($"Message [{eventArgs.DeliveryTag}] was not acked!");
    channel.BasicReturn += (sender, eventArgs) =>
    {
        string message = System.Text.Encoding.UTF8.GetString(eventArgs.Body);

        Console.WriteLine($"Message [{message}] rejected because it could not be routed to exchange [{eventArgs.Exchange}] with routing key [{eventArgs.RoutingKey}]");
    };

    byte[] messageBody = System.Text.Encoding.UTF8.GetBytes("Test message");

    channel.BasicPublish(
        exchange: string.Empty,
        routingKey: routingKey,
        mandatory: true, // Caso a flag mandatory esteja true, podemos receber o Basic.Return no caso da mensagem não ser roteada
        basicProperties: null,
        body: messageBody);

    channel.WaitForConfirms(); // Mais detalhes sobre esse abaixo
}
```

Quando o *publisher confirms* for habilitado através do método do *client* `channel.ConfirmSelect()`, o canal precisa ter certeza de que a mensagem não foi perdida. Isso pode ser feito através dos seguintes métodos:

- `channel.WaitForConfirms()`: esse método espera que todos os *acks* (ou *nacks*) tenham sido recebidos. Possui um overload que permite que um *timeout* seja especificado e que expõe um parâmetro *out* que indica se o timeout foi atingido. 

- `channel.WaitForConfirmsOrDie()`: esse método se comporta de maneira semelhante ao anterior, com as seguintes diferenças:
    - se o timeout dos *acks* foi atingido ou se um *nack* for recebido, ele fecha o *canal* e dispara uma `IOException`

### Resumindo *publisher confirms*

Utilize as confirmações para garantir que a mensagem foi entregue à algum consumidor. Elas são uma extensão do protocolo AMQP que permitem verificar de maneira relativamente leve que a mensagem foi entregue. Esse tipo de abordagem é extremamente útil para sistemas críticos aonde o *fire and forget* não pode ser implementado pois é preciso garantir a entrega de mensagens. Com *publisher confirms* estabelecemos uma forma de comunicação indireta, assíncrona e confiável entre produtores e consumidores.

---

## Lidando com possíveis perdas de mensagens

Conforme visto no *post anterior* é possível persistir mensagens em disco para garantir que elas sobrevivam à um restart do broker. Isso não garante, contudo, que a entrega das mensagens seja 100% garantida. A documentação oficial tem um [excelente exemplo](https://rabbitmq.docs.pivotal.io/35/rabbit-web-docs/confirms.html) sobre isso:

- Um app *produtor* publica uma mensagem para uma fila com a propriedade `durable=true`;
- Outro app *consumidor* retira a mensagem, com `deliveryMode` persistente, da fila durável, mas não emite o *ack* pois ele é feito de forma manual e a mensagem não acabou de ser processada;
- O *broker* sofre uma falha e reinicia;
- O *consumidor* reconecta e volta a consumir mensagens;

O restart do broker faz ele perder a mensagem, mesmo com o *delivery mode* da mensagem marcado como *persistente* e com a fila *durável*. Nesse caso, o *publisher confirms* habilitado faria com que a mensagem não fosse perdida, uma vez que o *ack* emitido pelo *consumidor* para esta mensagem ainda não teria sido emitido ou não teria sido recebido pelo *produtor*.

---

## Trade-offs

---

## Resumo

Neste post vimos como a *flag* `mandatory` do comando `Basic.Publish` do AMQP pode ser útil para garantir o roteamento da mensagem e como a extensão **publisher confirms** do comando `Confirm.Select` podem ser úteis para garantias não só de entregas, mas também de processamentos de mensagens críticos em sistemas distribuídos que utilizam o RabbitMQ como mensageria. Com estes dois mecanismos em conjunto, conseguimos estabelecer uma comunicação assíncrona pra garantir *confiabilidade* na entrega e processamento de mensagens entre produtores e consumidores.

---

## Fontes:

### Livro

-  [RabbitMQ in Depth, by Gavin M. Roy - Manning Publications](https://www.manning.com/books/rabbitmq-in-depth)

> Recomendo este excelente livro para quem quiser ir à fundo desde a especificação do AMQP, como ela é implementada pelo RabbitMQ até o provisionamento de infra-estrutura através de clusters.

### Documentações, sites e blogs

- RabbitMQ Blog - Introducing Publisher Confirms: https://www.rabbitmq.com/blog/2011/02/10/introducing-publisher-confirms/
- Consumer Acknowledgements and Publisher Confirms: https://www.rabbitmq.com/confirms.html
- RabbitMQ Publisher Confirms and Returns Callback with/without Mandatory flag, by SHARAD GUPTA: https://outline.com/RwD8Ez
- The differente failures on BasicPublish, by Jack Vanlight: https://jack-vanlightly.com/blog/2017/3/10/rabbitmq-the-different-failures-on-basicpublish
- RabbitMQ.Client `ModelBase` implementation (C#): https://github.com/rabbitmq/rabbitmq-dotnet-client/blob/da219fe8846e608e1b3bcca484483015892ba5c7/projects/client/RabbitMQ.Client/src/client/impl/ModelBase.cs