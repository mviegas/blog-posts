---
title: "(pt-BR) Utilizando a API do RabbitMQ para criar Virtual Hosts"
publishdate: 2999-12-19
description: 
tags: ["dotnet", "rabbitmq", "distributedsystems", "microservices"]
cover_image:
canonical_url:
---

# (pt-BR) Utilizando a API do RabbitMQ para criar Virtual Hosts 

> *tl/dr*: Esse post é uma dica rápida sobre como verificar se um Virtual Host existe na nossa instância do RabbitMQ (e criar, caso não exista).

# O problema

Temos uma aplicação multi-tenant na qual tivemos que trabalhar com as exchanges/filas separadas por tenant. Pra isso, tivemos a idéia de ter um *virtual host* para cada tenant mas não queríamos ter o trabalho de inserir esses vhosts manualmente um por um no RabbitMQ.

# A solução

Utilizamos o *RabbitMQ.Client* para implementar nossa conexão com o Rabbit e logo pensei que lá haveria um método que fizesse isso. Hmm ... mas não há! Descobrimos, contudo, que um a API REST do RabbitMQ possui um endpoint que faz exatamente isso que queríamos.

Por isso, implementamos um método de extensão simples e rápido que verifica se o vhost especificado existe e, se não existir, cria.

```csharp
public static class RabbitMqExtensions
    {
        public static void EnsureVirtualHostExists(this ConnectionFactory connectionfactory)
        {
            var credentials = new NetworkCredential() { UserName = connectionfactory.UserName, Password = connectionfactory.Password };
            using (var handler = new HttpClientHandler { Credentials = credentials })
            using (var client = new HttpClient(handler))
            {
                var url = $"http://{connectionfactory.HostName}:15672/api/vhosts/{connectionfactory.VirtualHost}";

                var content = new StringContent("", Encoding.UTF8, "application/json");
                var result = client.PutAsync(url, content).Result;

                if ((int)result.StatusCode >= 300)
                    throw new Exception(result.ToString());
            }
        }
    }
```

# Melhorias

Há um problema de rede com criar uma instância do `HttpClient` a cada chamada. Existe um pool limitado de conexões que pode se esgotar facilmente no caso uma sobrecarga nas chamadas. O jeito correto de fazer isso, utilizar uma `HttpClientFactory` para lidar com o pool de conexões. Mais infos sobre isso [aqui](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/http-requests?view=aspnetcore-3.1). 
