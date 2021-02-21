---
title: "RabbitMQ: implementando uma conexão resiliente com Polly em C#"
description: 
publishdate: 2999-12-19
tags: ["dotnet", "rabbitmq", "distributedsystems", "microservices"]
---

# (pt-BR) RabbitMQ e .NET Core: implementando uma conexão resiliente com Polly

> *tl/dr:* Nesse post vou explicar

## Ciclo de vida de uma conexão no RabbitMQ

[Traduzindo da documentação oficial](https://www.rabbitmq.com/dotnet-api-guide.html#connection-and-channel-lifspan): *Conexões são feitas para terem vida longa. O protocolo por debaixo delas foi arquitetado e otimizado para que sejam de longa duração. Isso significa que abrir uma conexão nova por operação, como por exemplo uma mensagem publicada, é desencessário e fortemente desencorajado pois irá introduzir overheads e sobrecarga na rede.*

Bom, dito isso vamos em frente.

## Aprimorando o exemplo anterior

No post de [introdução ao RabbitMQ com .NET Core](https://dev.to/mviegas/pt-br-introducao-ao-rabbitmq-com-net-core-15oc) eu falei bastante sobre os conceitos básicos, mas mostrei também um exemplo de como implementar uma conexão básica com o broker.

Vamos então aprimorar a forma como fazemos isso no exemplo anterior.

## A palavra do momento: resiliência e porque ela importa

Bom, agora nossa aplicação possui uma conexão de longa duração com o Rabbit e não abre uma nova a cada chamada registrada. Porém, essa conexão está em cima de camada ~~super confiável~~ extremamente suscetível à falhas e variações: **a rede**. Pensando então em algumas coisas possíveis de acontecer:

- O Broker pode estar indisponível. Lembrando o [post anterior](https://dev.to/mviegas/pt-br-introducao-ao-rabbitmq-com-net-core-15oc), com um broker como elemento central numa arquitetura distribuída deve ter a disponibilidade garantida pois passa a ser uma peça chave da infra-estrutura. Contudo, algo pode correr que ele pode não estar acessível. Nesse caso é disparada uma `BrokerUnreachableException`.
- Nossa rede pode não ter mais sockets disponíveis para abrir uma nova conexão, pode não haver memória suficiente para isso, pode ocorrer alguma processamento bloqueante, alguma variação na topologia da rede que aborte uma operação no meio ... em todos esses casos seria disparada uma `SocketException`.

Pensando nesses dois casos, podemos implementar uma **política** que trate esses dois tipos de falhas e saiba como lidar caso algo dê errado. Para isso usamos a biblioteca [Polly](https://github.com/App-vNext/Polly). Que ferramenta incrível, é algo que todo desenvolvedor devia conhecer (na minha humilde opinião).

## Falhas na rede podem ser esporádicas, então esperamos e tentamos de novo

Primeiro devemos adicionar no nosso projeto o package do Polly. Basta na CLI na raiz do projeto: `dotnet add package Polly`
