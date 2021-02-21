---
title: Explorando System.Threading.Channels
description: "Este post tem como objetivo ser uma explicação prática da API System.Threading.Channels e como ela pode ser útil para implementar comunicação entre Tasks seguindo padrões pub/sub."
date: 2020-05-17
language: pt
tags: ["dotnet", "pubsub" ]
---

# *tl/dr* 

Este post tem como objetivo ser uma explicação prática da API System.Threading.Channels e como ela pode ser útil para implementar comunicação entre Tasks seguindo padrões pub/sub.

---

# Produtores x Consumidores

[Neste post](https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/) do blog da Microsoft é levantada a motivação pra criação da lib System.Threading.Channels. São recorrentes, até mesmo no dia a dia, os cenários de se produzir algo por uma ou mais partes e que esse algo vai posteriormente ser consumido por uma ou mais outras partes. E ainda que a produção e o consumo sejam feito de maneiras independentes por cada uma dessas partes, ambas precisam estar conectadas uma(s) à(s) outra(s) para que isso aconteça. Por exemplo, uma encomenda enviada por um remetente, para ser entregue a um destinatário, precisa passar por todo um processo logístico até que isso aconteça. Esse processo pode ser abstraído como um *canal* entre as duas partes interessadas, e esse canal que permite o recebimento.

É justamente uma abstração dessa conexão, entitulada de **channel** que é criada com a lib [System.Threading.Channels](https://www.nuget.org/packages/System.Threading.Channels). Para usá-la, basta instalar um package nuget, que é compatível com o .NET Standard(>= 1.3), .NET Core (>= 3.1) e até mesmo com o NET Framework (>= 4.6).

---

# Dissecando um channel

## Abrindo channels

Um **channel** pode ser visto, portanto, como uma estrutura de dados básica pra estabelecer uma conexão entre produtores e consumidores. É uma estrutura genérica e que pode ser criada através da classe `System.Threading.Channels.Channel<T>` pelos seguintes métodos, que são static factories:

- `CreateBounded<T>(int capacity)`: cria um *bounded channel*, que é um *canal* que tem uma capacidade de objetos (mensagens) definida. Se um *channel* é criado desta forma, significa que, uma vez que ele atingiu sua capacidade máxima, ele só poderá receber um novo objeto depois que outro objeto previamente colocado nele por um *writer* seja consumido por um *reader*.
- `Channel<T> CreateBounded<T>(BoundedChannelOptions options)`: tem exatamente o mesmo comportamento da factory acima, com o adicional da possibilidade de definição de opções do canal. Na data de escrita deste post, essas opções setam não só a capacidade, mas também como o *channel* vai se comportar no caso de ter chegado no topo de sua capacidade. Isso é definido através da propriedade `FullMode`, um enumerado com quatro opções:
  - `Wait`: espera que o canal tenha espaço livre para completar uma operação de escrita.
  - `DropNewest`: descarta o item mais novo já escrito mas ainda não lido no canal, para abrir espaço para o que acabou de chegar.
  - `DropOldest`: de maneira semalhante à operação acima, porém com o item mais antigo já escrito mas ainda não lido no canal.
  - `DropWrite`: descarta outro item que esteja sendo escrito no canal.
- `Channel<T> CreateUnbounded<T>()`: cria um *unbounded channel* - um canal sem capacidade definida, aonde o céu (na verdade, a quantidade de memória RAM disponível) é o limite. Segundo testes de benchmark apresentados [por este blog](https://ndportmann.com/system-threading-channels/), *unbounded channels* são mais performáticos. Contudo, criar canais dessa forma pode ser perigoso uma vez que eles não tem limitação e podem consumir toda a memória disponível. Por esta razão, é dito que eles são menos seguros que os seus irmãos apresentados anteriormente. Deve ser analisado o tradeoff de segurança x performance, caso o nível de requisitos seja crítico a este ponto. Caso seja possível garantir a capacidade máxima de maneira externa, pode valer o risco.
- `Channel<T> CreateUnbounded<T>(UnboundedChannelOptions options)`: tem exatamente o mesmo comportamento da factory acima, com o adicional de definição de opções do canal. E esta classe tem uma propriedade que é bem interessante:
  - `AllowSynchronousContinuations`: de acordo com a [doc oficial](https://docs.microsoft.com/en-us/dotnet/api/system.threading.channels.channeloptions.allowsynchronouscontinuations?view=netcore-3.1#System_Threading_Channels_ChannelOptions_AllowSynchronousContinuations), essa propriedade tem uma comportamento bem interessante. Ela permite, de certaforma, que um *produtor* vire, temporariamente, um *consumidor*. Como? Vamos entrar nesse detalhe mais abaixo após entender como produtores (*writers*) e consumidores (*readers*) se comportam.

Ambas as classes de opções de *unbounded* e *bounded channels* herdam de `ChannelOptions` que, por sua vez, possui duas propriedades:
 - `SingleWriter`: tem como default false. Se setada como true, é garantida que acontecerá apenas uma operação de escrita por vez. Por padrão, não existe essa constraint, e a própria estrutura da api de channels pode otimizar algumas operações de escrita internamente.
 - `SingleReader`: o mesmo comportamento da operação acima, porém com operações de leitura no canal.

No exemplo abaixo, criamos um *bounded channel* que receberá objetos do tipo `int` com capacidade máxima de 1.

```csharp
var channel = System.Threading.Channels.Channel.CreateBounded<int>(new BoundedChannelOptions(1)
{
    FullMode = BoundedChannelFullMode.Wait
});
```

## Encerrando channels

Um canal pode ser encerrado através de um recurso chamado *completion*. Este recurso é controlado sempre pelos produtores (*writers*) e pode ser ativado através dos métodos `Complete` e `TryComplete`, que podem inclusive receber uma exceção para indicar que o *channel* foi encerrado devido à um erro.

Durante uma operação de leitura, os consumidores (*readers*) podem, por sua vez, verificar se um canal foi encerrado chamando `channel.Completion.IsCompleted`.

## Produzindo e consumindo mensagens

Mensagens podem ser produzidas no canal através de um `Writer` e lidas por um `Reader`, propriedades da classe `Channel<T>`. Isso pode através dos seguintes métodos:

1) `TryWrite/TryRead`: tentam escrever/ler uma mensagem de maneira síncrona no canal. Caso não seja possível, pelas opções definidias, a mensagem é descartada e o método retorna `false`. Caso contrário, retorna `true`. No exemplo abaixo, escrevemos no console se uma mensagem não pôde ser escrita no canal. Isso ficará mais claro no exemplo inteiro que daremos um pouco mais à frente, aonde colocamos um delay entre a leitura e a escrita pra simular cenários em que o canal esteja funcionando na capacidade total e, por consequência e a depender de como foi configurado, esteja impedido de receber novas escritas de mensagens.

```csharp
for (int i = 0; ; i++)
{
    if (!channel.Writer.TryWrite(i))
    {
        Console.WriteLine($"Dropping {i}");
    }

    await Task.Delay(1000);
}
```

2) `WaitToWriteAsync/WaitToReadAsync`: retornam uma `ValueTask<bool>` e indicam se o canal está disponível para escrever/ler mensagens, baseados nas opções e capacidade definidas. Por padrão, estes métodos não dispararão uma exceção caso o canal já esteja encerrado, a não ser que uma exceção seja passada por parâmetro nos métodos `Complete` e `TryComplete`, vistos acima. No exemplo do código abaixo, o consumidor (*reader*) permanece ativo enquanto o loop for verdadeiro, e isso acontecerá indefinidamente a não ser que o canal seja encerrado. Enquanto o canal não for encerrado, mas também não chegarem mensagens, ele ficará inativo. A implementação conjunta com o TryRead é interessante pelo seguinte fato: num cenário onde múltiplos consumidores competem por uma mesma mensagem, e a produção dessas mensagens é feita de forma aleatória, a mensagem que foi sinalizada como disponível para todos os consumidores pode ser consumida mais rapidamente  por um outro consumidor concorrente.

```csharp
while (await channel.Reader.WaitToReadAsync())
{
    if (channel.Reader.TryRead(out string msg))
    {                    
        Console.WriteLine(msg);
    }
    else
    {
        Console.WriteLine($"Message already {msg} consumed");
    }
}
```

3) `WriteAsync/ReadAsync`: são a implementação assíncrona dos métodos `TryRead`/`TryWrite`, com um importante adendo de que ambos disparam uma exceção do tipo `ChannelClosedException` se o canal já tiver sido encerrado por um *writer*. Para se evitar essa situação, é preferível usar os métodos `WaitTo`.

```csharp
private static async Task ReadAsync(Channel<int> channel, int delay)
{
    while (true)
    {
        var message = await channel.Reader.ReadAsync();
        Console.WriteLine($"Readed {message}");
        await Task.Delay(delay).ConfigureAwait(false);
    }
    // An exception will be thrown when the channel is closed
}
```


### Consumindo todas as mensagens de uma vez

Além dos pares de métodos acima, temos ainda o `ReadAllAsync` que cria um `IAsyncEnumerable` de todos os dados disponíveis para leitura no canal e nos permite iterar de maneira assíncrona sobre eles.


### Voltando ao AllowSynchronousContinuations

Como dito anteriormente, a propriedade `AllowSynchronousContinuations` permite com que um produtor vire temporariamente um consumidor. Vamos pensar no seguinte exemplo: um consumidor chama uma operação de leitura num canal sem mensagens, internamente esse canal cria algo como um callback que será chamado quando alguma mensagem for escrita nele e, por default, isso é feito de maneira assíncrona enfileirando a invocação desse callback para ser executada em uma thread diferente da thread do produtor. 

Quando setamos essa propriedade como `true`, estamos dizendo ao canal que esse callback pode ser feito de maneira síncrona, ou seja, a mesma thread que escreveu a mensagem irá consumí-la. Isso pode ser uma vantagem em termos de performance, mas deve ser usada com cuidado em alguns cenários. Por exemplo, se algum tipo de `lock` estiver sendo usado na escrita, a chamada do callback de leitura pode ocorrer com esse lock ativo, o que pode gerar comportamentos não-desejados no seu código.

---

# Exemplos

Os exemplos abaixo podem ser encontrados [neste repo](https://github.com/mviegas/ChannelsDemo). Nele temos implementações básicas de um produtor e um consumidor interagindo através de um *channel* de maneira assíncrona e não-bloqueante.

## Escrevendo num canal com a capacidade máxima

No exemplo abaixo, setamos velocidades diferentes de leitura e escrita para que a escrita tente acontecer em momentos em que o canal estiver em sua capacidade máxima.

```csharp
private static async Task ChannelOutOfCapacityExample()
{
    // Setting a read delay bigger than a wrote delay, so that we can see what happens when channels are "full"
    const int readDelay = 500;

    const int writeDelay = 100;

    // Creating a bounded channel with capacity of 1 
    var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(1)
    {
        // Setting this property we say that when the channel is full and another item is dispatched to it, it should wait until the current item is processed to process the next
        FullMode = BoundedChannelFullMode.Wait
    });

    // Calling Task.Run so that the Channel.Writer executes in a different synchronization context than the Channel.Reader
    _ = Task.Run(async () =>
    {
        for (int i = 0; ; i++)
        {
            if (!channel.Writer.TryWrite(i))
            {
                ExtendedConsole.WriteLine($"Dropping {i}", ConsoleColor.Red);
            }

            await Task.Delay(writeDelay).ConfigureAwait(false);
        }
    });

    while (true)
    {
        var message = await channel.Reader.ReadAsync().ConfigureAwait(false);

        ExtendedConsole.WriteLine($"Readed {message}", ConsoleColor.Green);

        await Task.Delay(readDelay).ConfigureAwait(false);
    }
}
```

## Encerrando um canal sem exceções pelo consumidor

No exemplo abaixo exemplificamos o uso do WaitToReadAsync para mostrar como exceções podem ser evitadas no *completion* do canal.

```csharp
private static async Task ChannelCompletedWithoutExceptionRaised()
{
    var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(1)
    {
        FullMode = BoundedChannelFullMode.Wait
    });

    _ = Task.Run(async () =>
    {
        for (int i = 0; ; i++)
        {
            await channel.Writer.WriteAsync(i);
            ExtendedConsole.WriteLine($"Writing {i}", ConsoleColor.Blue);

            if (i == 10)
            {
                ExtendedConsole.WriteLine($"Writer: completing channel after 10 executions", ConsoleColor.Yellow);
                channel.Writer.TryComplete();
            }
        }
    });

    // Using WaitToRead, no exception is raised when channel is completed, unless it is explicit passed on completion
    while (await channel.Reader.WaitToReadAsync(default).ConfigureAwait(false))
    {
        if (channel.Reader.TryRead(out int msg))
        {
            ExtendedConsole.WriteLine($"Readed {msg}", ConsoleColor.Green);
        }
        else
        {
            Console.WriteLine($"Message already {msg} consumed");
        }
    }
}
```


---

## Consumidores concorrentes 

No exemplo abaixo, temos um canal com um produtor e três consumidores concorrentes. Dessa forma, quando um consumidor é ativado pelo `WaitToReadAsync`, por existirem outros consumidores para o mesmo canal, a mensagem que ativou o método já pode ter sido consumida por outro concorrente, o que faria o `TryRead` retornar false.

```csharp
private static async Task ChannelWithCompetingConsumers()
{
    var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(1)
    {
        FullMode = BoundedChannelFullMode.Wait
    });

    _ = Task.Run(async () =>
    {
        for (int i = 0; ; i++)
        {
            await channel.Writer.WriteAsync(i);
            ExtendedConsole.WriteLine($"Writing {i}", ConsoleColor.Blue);

            if (i == 50)
            {
                ExtendedConsole.WriteLine($"Writer: completing channel after 50 executions", ConsoleColor.Yellow);
                channel.Writer.TryComplete();
            }
        }
    });

    var firstConsumer = new Consumer<int>(channel.Reader);
    var secondConsumer = new Consumer<int>(channel.Reader);
    var thirdConsumer = new Consumer<int>(channel.Reader);

    await Task.WhenAll(
        firstConsumer.ConsumeAsync(default),
        secondConsumer.ConsumeAsync(default),
        thirdConsumer.ConsumeAsync(default)).ConfigureAwait(false);
}


private class Consumer<T>
{
    private readonly ChannelReader<T> _reader;
    public Guid Id { get; }
    private readonly Random _random = new Random(100);

    public Consumer(ChannelReader<T> reader)
    {
        Id = Guid.NewGuid();
        _reader = reader;
    }

    public async Task ConsumeAsync(CancellationToken cancellationToken)
    {
        // Using WaitToRead, no exception is raised when channel is completed, unless it is explicit passed on completion
        while (await _reader.WaitToReadAsync(cancellationToken))
        {
            if (_reader.TryRead(out T msg))
                ExtendedConsole.WriteLine($"ID {Id} === Readed {msg}", ConsoleColor.Green);
            else
                ExtendedConsole.WriteLine($"ID {Id} === Consumer awoken but message already consumed", ConsoleColor.Yellow);

            await Task.Delay(_random.Next(500));
        }
    }
}
```

# Conclusão

Como visto, **channels** são abstrações bem poderosas pra mecanismos de pub/sub de maneira assíncrona e não-bloqueante com aplicações .NET (Core e Framework). Sua implementação tem como foco trabalhar em cenários concorrentes com excelentes performance e flexibilidade e são um recurso bem interessante, principalmente pra processamentos feitos de maneira assíncrona através da comunicação entre várias *tasks*.

---

# Referências

- Working with Channels in .NET: https://www.youtube.com/watch?v=gT06qvQLtJ0

- An Introduction to System.Threading.Channels: https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/

- An Introduction to System.Threading.Channels (mesmo título, outro post): https://www.stevejgordon.co.uk/an-introduction-to-system-threading-channels

- Exploring System.Threading.Channels: https://ndportmann.com/system-threading-channels/