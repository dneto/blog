+++
date = '2025-05-24T09:00:00-03:00'
draft = false
title = 'Contextos no Go'
layout = 'post'
tags = ['go', 'programação', 'concorrência']
author = 'Demétrio Neto'
description = 'Entenda de forma leve e prática como o pacote context funciona no Go — com analogias, exemplos e boas práticas.'
+++

## Introdução

Uma das responsabilidades que possuo onde trabalho atualmente é ensinar e
orientar pessoas menos experientes, e uma das principais dúvidas que recebo
relacionadas a Go é sobre os contextos.

> [!QUOTE]
> Mas eu não aguento mais passar isso em praticamente toda chamada que faço.
> Outras linguagens não têm isso, por que eu tenho que me preocupar?
>
> \- Pessoa desenvolvedora vendo contextos em todo lugar

Para entender a utilidade dos contextos, precisamos estar cientes de que
[concorrência](https://en.wikipedia.org/wiki/Concurrency_(computer_science))
é algo comum no desenvolvimento atual, **mesmo que não percebamos**.

Novas tarefas podem surgir das **mais diversas formas**:
[processos](https://en.wikipedia.org/wiki/Process_(computing)),
[threads](https://en.wikipedia.org/wiki/Thread_(computing)), _goroutines_
(ou [green threads](https://en.wikipedia.org/wiki/Green_thread)). Além de
surgir, elas também podem — **e serão** — interrompidas.

O Go nos oferece uma forma de sinalizar que um conjunto de tarefas precisa ser cancelado e também de tratar esse sinal no código de um sistema.

## Interrupções no dia a dia

Nosso cotidiano é repleto de situações que não saem como o planejado. Muitas
vezes, precisamos ter um plano B ou parar para pensar em como lidar com essas
mudanças. Nos sistemas que construímos, isso não é diferente — exceto que os
computadores fazem exatamente o que pedimos, sem a capacidade de se adaptar
sozinhos.

Para ilustrar essa ideia, vamos a uma analogia:

> [!ANALOGY] Analogia
> Imagine que você vai almoçar em um restaurante. Ao se sentar, escolhe um
> prato do cardápio, chama o garçom e faz seu pedido.
>
> O garçom anota o pedido e o envia para a cozinha, onde os cozinheiros
> começam a prepará-lo.
>
> Agora, imagine que, no meio disso, você recebe uma ligação urgente e
> precisa sair imediatamente. Você cancela o pedido com o garçom.
>
> O que acontece com o prato que estava sendo preparado? Isso depende do
> funcionamento do restaurante: ele pode descartar tudo o que já foi feito,
> ou tentar reaproveitar os ingredientes que ainda estão bons.

Assim como o restaurante precisou lidar com o pedido cancelado, em sistemas
concorrentes também há momentos em que precisamos interromper uma tarefa em
andamento.

No exemplo, você percebeu que precisava sair e não voltaria, então cancelou
o pedido com o garçom — que, por sua vez, comunicou o cancelamento à cozinha.

### O que isso tem a ver com meu código?

Além do exemplo do restaurante, podemos pensar em situações mais próximas da
realidade de quem desenvolve: pode ser o download de um arquivo grande, uma
requisição para uma API REST, uma consulta ao banco de dados ou a execução de
qualquer tarefa que pode levar muito tempo para completar.

Como você faria para cancelar algo, se fosse necessário? E como notificaria esse
cancelamento em efeito cascata para todas as operações relacionadas?

Às vezes, uma simples sinalização é suficiente. Em outros casos, como no
restaurante, o cancelamento precisa ser propagado por diferentes partes do
sistema — e pode ter consequências importantes que precisam ser tratadas
adequadamente.

Cada linguagem, biblioteca ou framework oferece sua própria solução para lidar
tanto com o ciclo de vida quanto com o tempo de vida de fluxos e operações. Os
desenvolvedores do Go enxergaram essa necessidade — e é aí que entra o pacote
`context`.

Mas antes de falar sobre contextos, vale lembrar: o Go foi criado com a
filosofia de que **ser explícito é melhor do que esconder complexidade por trás
de abstrações implícitas**.

>[!QUOTE]
> Explícito é melhor que implícito
>
> \- Item 2 do [Zen of Python](https://peps.python.org/pep-0020/)

Apesar das coisas nem sempre serem tão explícitas assim no Python, esse é um bom
conselho a ser seguido para qualquer linguagem. Inclusive, recomendo também a
leitura do [Zen of Go](https://dave.cheney.net/2020/02/23/the-zen-of-go).

## Entrando no contexto

Agora que entendemos por que é importante poder interromper tarefas, vamos ver
como o Go nos ajuda a fazer isso de forma estruturada com o pacote `context`.

O pacote `context` define um padrão para lidar com cenários em que sinais de
cancelamento, timeouts ou deadlines precisam ser propagados ao longo da
execução. Ele permite que fluxos sejam encerrados de forma controlada — o que
costumamos chamar de _**graceful shutdown**_.

### A interface `context.Context`

Vamos começar olhando a [documentação da interface `context.Context`](https://pkg.go.dev/context#Context). Na descrição do tipo, vemos:

> [!QUOTE] Citação
> _A Context carries a deadline, a cancellation signal, and other values across API boundaries._\
> _Context's methods may be called by multiple goroutines simultaneously._

> [!TRANSLATION] Tradução
> _Um contexto carrega um prazo, um sinal de cancelamento, e outros valores através dos limites da API._\
> _Os métodos do tipo Context podem ser chamados simultaneamente por múltiplas goroutines_

### Como cancelar contextos?

Se você foi curioso ou curiosa e também deu uma olhada na própria definição da
interface, possivelmente notou que não existe um método `Cancel()` ou algo
parecido. Ou seja, um contexto não tem capacidade de ativar um sinal de
cancelamento para ele mesmo.

### Por que os contextos não podem cancelar a si próprios?

Para explicar o motivo, vamos voltar ao exemplo do restaurante: imagine que o
garçom ou outro cliente pudesse cancelar seu pedido, ou pior, todos os pedidos.
Você também pode pensar em uma tarefa que inicia várias _goroutines_ em paralelo
para realizar o download de vários arquivos — a exposição de uma forma de
cancelar dentro do próprio contexto poderia possibilitar o cancelamento de
outros downloads iniciados pela mesma tarefa. Seria uma tragédia! 😟

Logo, a existência da possibilidade de cancelar um contexto em qualquer escopo
seria terrivelmente perigosa e poderia impactar diversos fluxos de forma
imprevisível. Então, caso seja necessário ter um controle fino sobre o tempo de
vida de um determinado fluxo da sua aplicação, a recomendação é criar uma nova
instância do `context.Context`. O pacote `context` já oferece algumas formas
para criar contextos — vamos dar uma olhada nelas?

## Criando novos contextos

É importante lembrar que, exceto pelos métodos `context.Background()` e
`context.TODO()`, todos as formas de criação de um contexto exigem um _contexto-pai_.

Sempre que um contexto-pai for cancelado, todos seus filhos também serão cancelados.

Nessa seção teremos alguns exemplos mais completos que irão utilizar o código abaixo, esses exemplos também contam com um link para execução no [Go Playground](https://play.golang.org).

### Estrutura dos exemplos

Para alguns dos exemplos seguir, decidi seguir a _vibe_ de restaurante e criei
duas `structs`.

A `restaurant` representando o próprio restaurante, com o metodo `order`
para "receber" os pedidos e encaminhá-los a cozinha:

```go
type restaurant struct {
    kitchen kitchen
}

func (r restaurant) order(ctx context.Context, dish string) {
    fmt.Printf("restaurante: preparando o pedido %q\n", dish)
    r.kitchen.cook(ctx, dish)
    <-ctx.Done() // espera o contexto ser cancelado
    fmt.Printf("restaurante: pedido %q cancelado: %s\n", dish, context.Cause(ctx))
}
```

A outra, `kitchen`, representa a cozinha — e tem o método `cook` representando o
ato de cozinhar o prato:

```go
type kitchen struct{}

func (k kitchen) cook(ctx context.Context, dish string) {
    fmt.Printf("cozinha: fingindo que estamos fazendo %q\n", dish)
    <-ctx.Done() // espera o contexto ser cancelado
    fmt.Printf("cozinha: parando de fazer %q: %s\n", dish, context.Cause(ctx))
}
```

Note que nesse exemplo estamos usando o método `ctx.Done()`, e a função `context.Cause(ctx)`, vamos falar mais sobre elas na seção [lidando com o cancelamento](#lidando-com-o-cancelamento)

### Sem possibilidade de cancelamento

Decidi organizar as funções de criação de contexto pela possibilidade ou não de
cancelamento. Vamos iniciar pelas formas que não oferecem nenhum tipo de
mecanismo para cancelamento.

#### Criando contextos básicos — `context.Background`

O `context.Background` cria um novo contexto vazio, ou seja, não possui prazos, nem guarda valores — e não pode ser cancelado. Pode ser criado no início da aplicação, ou em testes, por exemplo.

```go
ctx := context.Background()
```

#### Criando contextos provisórios - `context.TODO`

O `context.TODO` pode ser usado quando não se tem certeza de qual outra opção
deve ser usada. A intenção dele é ser apenas um _placeholder_ e deve ser substituído.

Assim como o `context.Background`, ele não pode ser cancelado.

```go
ctx := context.TODO()
```

#### Contextos que carregam valores — `context.WithValue`

Também existe a função `context.WithValue`, que permite a criação de um contexto
com valores armazenados internamente.

[Exemplo completo no Go Playground](https://go.dev/play/p/ks4RjpRvYKM)

```go
func main() {
    iceCreamPlace := restaurant{kitchen: kitchen{}}

    // Cria um contexto com valor
    ctx := context.WithValue(context.Background(), "orderId", "1")
    dish := "sorvete de cebola"
    fmt.Printf("cliente: pedindo %q\n", dish)

    iceCreamPlace.order(ctx, dish)
}
```

```text
cliente: pedindo "sorvete de cebola"
restaurante: recebendo pedido de "sorvete de cebola"
cozinha: preparando "sorvete de cebola" (pedido #1)
```

> [!CAUTION] Atenção
> Recomendo **bastante cautela** ao utilizar
> contextos dessa forma, embora seja muito útil para passar agentes de métricas,
> tracing ou dados de um request — como um request id — entre as diferentes camadas,
> o abuso dessa opção pode causar problemas de clareza no código.
>
> Essa funcionalidade não deve ser utilizada como um dicionário genérico global.

#### Criando contextos derivados sem cancelamento — `context.WithoutCancel`

> [!WARNING] Disponível a partir do Go 1.21

O `context.WithoutCancel` cria um novo contexto a partir de um contexto-pai, mas continuará ativo, mesmo que o original tenha sido cancelado.

[Exemplo completo no Go Playground](https://go.dev/play/p/74cJvUW3q4F)

```go
func main() {
    iceCreamPlace := restaurant{kitchen: kitchen{}}

    // Cria um contexto com cancelamento
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    // Cria um contexto derivado que ignora o cancelamento do pai
    ctx2 := context.WithoutCancel(ctx)

    dish := "sorvete de cebola"
    fmt.Printf("cliente: pedindo %q\n", dish)

    go func() {
        iceCreamPlace.order(ctx2, dish)
    }()

    fmt.Println("cliente: esperando o pedido ficar pronto")
    time.Sleep(1 * time.Second)
    fmt.Printf("cliente: cancelando %q\n", dish)
    cancel() // Cancela o contexto pai, mas ctx2 não será cancelado

    time.Sleep(2 * time.Second)
    fmt.Printf("cliente: verificando status do pedido %q\n", dish)
}
```

### Com possibilidade de cancelamento

#### Criando contextos com cancelamento — `context.WithCancel` e `context.WithCancelCause`

A função `context.WithCancel` possui dois retornos. O primeiro é o próprio
contexto, que deve ser repassado nas chamadas em que ele se faz necessário, já o
segundo é uma função de cancelamento que, ao ser chamada, irá cancelar o
contexto criado.

> [!WARNING]
> É sempre uma boa prática utilizar o `defer` nas funções de
> cancelamento, pois ajuda a evitar o vazamento de _goroutines_.

[Exemplo completo no Go Playground](https://go.dev/play/p/xfY8hceWaHC)

```go
func main() {
    iceCreamPlace := restaurant{kitchen: kitchen{}}

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    dish := "sorvete de cebola"
    fmt.Printf("cliente: pedindo %q\n", dish)

    go func() {
        iceCreamPlace.order(ctx, dish)
    }()

    fmt.Println("cliente: esperando o pedido ficar pronto")
    time.Sleep(1 * time.Second)
    fmt.Printf("cliente: cancelando %q\n", dish)
    cancel()
    time.Sleep(1 * time.Second)
}
```

```text
cliente: pedindo "sorvete de cebola"
cliente: esperando o pedido ficar pronto
restaurante: preparando o pedido "sorvete de cebola"
cozinha: fingindo que estamos fazendo "sorvete de cebola"
cliente: cancelando "sorvete de cebola"
cozinha: parando de fazer "sorvete de cebola": context canceled
restaurante: pedido "sorvete de cebola" cancelado: context canceled
```

Também é possível evidenciar a causa do cancelamento utilizando a função
`context.WithCancelCause`.

[Exemplo completo no Go Playground](https://go.dev/play/p/rUI_qSkZTsF)

```go
func main() {
    iceCreamPlace := restaurant{kitchen: kitchen{}}

    ctx, cancel := context.WithCancelCause(context.Background())
    defer cancel(nil) // retorna o erro original: "context canceled"
    dish := "sorvete de cebola"
    fmt.Printf("cliente: pedindo %q\n", dish)

    go func() {
        iceCreamPlace.order(ctx, dish)
    }()

    fmt.Println("cliente: esperando o pedido ficar pronto")
    time.Sleep(1 * time.Second)
    cause := errors.New("cancelado pelo cliente")
    cancel(cause)
    time.Sleep(1 * time.Second)
}
```

```text
cliente: pedindo "sorvete de cebola"
cliente: esperando o pedido ficar pronto
restaurante: preparando o pedido "sorvete de cebola"
cozinha: fingindo que estamos fazendo "sorvete de cebola"
cozinha: parando de fazer "sorvete de cebola": cancelado pelo cliente
restaurante: pedido "sorvete de cebola" cancelado: cancelado pelo cliente
```

#### Com prazo de validade absoluto — `context.WithDeadline` e `context.WithDeadlineCause`

A `context.WithDeadline`, que recebe um `time.Time` e irá cancelar o
contexto automaticamente após **a data** informada.

[Exemplo completo no Go Playground](https://go.dev/play/p/7MiJmFFW1wl)

```go
func main() {
    iceCreamPlace := restaurant{kitchen: kitchen{}}

    // Define um deadline para 2 segundos no futuro
    deadline := time.Now().Add(2 * time.Second)
    ctx, cancel := context.WithDeadline(context.Background(), deadline)
    defer cancel()
    dish := "sorvete de cebola"
    fmt.Printf("cliente: pedindo %q\n", dish)

    go func() {
        iceCreamPlace.order(ctx, dish)
    }()

    fmt.Println("cliente: esperando o pedido ficar pronto")
    time.Sleep(3 * time.Second) // espera mais do que o deadline
    fmt.Printf("cliente: verificando status do pedido %q\n", dish)
}
```

```text
cliente: pedindo "sorvete de cebola"
cliente: esperando o pedido ficar pronto
restaurante: preparando o pedido "sorvete de cebola"
cozinha: fingindo que estamos fazendo "sorvete de cebola"
cozinha: parando de fazer "sorvete de cebola": context deadline exceeded
restaurante: pedido "sorvete de cebola" cancelado: context deadline exceeded
cliente: verificando status do pedido "sorvete de cebola"
```

Também podemos informar a causa do cancelamento trocando `context.WithDeadline`
por `context.WithDeadlineCause`:

[Exemplo completo no Go Playground](https://go.dev/play/p/DDNDSTnJcEM)

```go
func main() {
    iceCreamPlace := restaurant{kitchen: kitchen{}}

    deadline := time.Now().Add(2 * time.Second)
    cause := errors.New("hora de levar minha avó pra aula de judô")
    ctx, cancel := context.WithDeadlineCause(context.Background(), deadline, cause)
    defer cancel()

    // restante do código
}
```

```text
cliente: pedindo "sorvete de cebola"
cliente: esperando o pedido ficar pronto
restaurante: preparando o pedido "sorvete de cebola"
cozinha: fingindo que estamos fazendo "sorvete de cebola"
cozinha: parando de fazer "sorvete de cebola": hora de levar minha avó pra aula de judô
restaurante: pedido "sorvete de cebola" cancelado: hora de levar minha avó pra aula de judô
cliente: verificando status do pedido "sorvete de cebola"
```

#### Com prazo de validade relativo — `context.WithTimeout` e `context.WithTimeoutCause`

A `context.WithTimeout`, que recebe um `time.Duration` e irá cancelar o
contexto após o **período** informado.

[Exemplo completo no Go Playground](https://go.dev/play/p/JSwblzhicpx)

```go
func main() {
    iceCreamPlace := restaurant{kitchen: kitchen{}}

    // Define um timeout de 2 segundos
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()
    dish := "sorvete de cebola"
    fmt.Printf("cliente: pedindo %q\n", dish)

    go func() {
        iceCreamPlace.order(ctx, dish)
    }()

    fmt.Println("cliente: esperando o pedido ficar pronto")
    time.Sleep(3 * time.Second) // espera mais do que o timeout
    fmt.Printf("cliente: verificando status do pedido %q\n", dish)
}
```

```text
cliente: pedindo "sorvete de cebola"
cliente: esperando o pedido ficar pronto
restaurante: preparando o pedido "sorvete de cebola"
cozinha: fingindo que estamos fazendo "sorvete de cebola"
cozinha: parando de fazer "sorvete de cebola": context deadline exceeded
restaurante: pedido "sorvete de cebola" cancelado: context deadline exceeded
cliente: verificando status do pedido "sorvete de cebola"
```

>[!TIP] Note que a saída é igual a do exemplo com `context.WithDeadline`

Aqui também é possível informar uma causa:

[Exemplo completo no Go Playground](https://go.dev/play/p/bH3kwK7Nic0)

```go
func main() {
    iceCreamPlace := restaurant{kitchen: kitchen{}}

    // Define um timeout de 2 segundos
    cause := errors.New("desisti de esperar")
    ctx, cancel := context.WithTimeoutCause(context.Background(), 2*time.Second, cause)
    defer cancel()
    dish := "sorvete de cebola"
    fmt.Printf("cliente: pedindo %q\n", dish)

    go func() {
        iceCreamPlace.order(ctx, dish)
    }()

    fmt.Println("cliente: esperando o pedido ficar pronto")
    time.Sleep(3 * time.Second) // espera mais do que o timeout
    fmt.Printf("cliente: verificando status do pedido %q\n", dish)
}
```

```text
cliente: pedindo "sorvete de cebola"
cliente: esperando o pedido ficar pronto
restaurante: preparando o pedido "sorvete de cebola"
cozinha: fingindo que estamos fazendo "sorvete de cebola"
cozinha: parando de fazer "sorvete de cebola": desisti de esperar
restaurante: pedido "sorvete de cebola" cancelado: desisti de esperar
cliente: verificando status do pedido "sorvete de cebola"
```

### Criando seu próprio contexto

Por se tratar de uma interface, você pode criar sua própria implementação.
Pessoalmente não recomendo seguir por esse caminho, pois nesses meus quase 10
anos de Go eu ainda não vi nenhum cenário que as interfaces fornecidas pela
biblioteca padrão não foram suficientes.

## Lidando com o cancelamento

>[!COMMENT] _Não, não estamos falando de redes sociais_

Você pode ativamente verificar se o contexto já foi cancelado, essa abordagem é
útil para impedir que o fluxo prossiga. A forma mais comum é realizar a
verificação ao início da função, mas em alguns momentos pode ser importante
verificar ao final, para garantir que o fluxo não irá continuar.

### Verificando se o contexto já foi cancelado — `ctx.Err`

Você pode simplesmente verificar se o retorno de `ctx.Err()` é _não-nulo_. A
função retornará `nil` caso o contexto ainda não tenha sido cancelado e algum
erro caso o cancelamento tenha ocorrido.

```go
// verificação antes do processamento ocorrer
if ctx.Err() != nil {
    return fmt.Errorf("falha antes de iniciar: %w", ctx.Err())
}

// implementação do código

// verificação após o processamento ter ocorrido
if ctx.Err() != nil {
    return fmt.Errorf("falha após a execução: %w", ctx.Err())
}
```

### Identificando a causa do cancelamento — `context.Cause`

> [!WARNING] Disponível a partir do Go 1.21

A partir do [Go 1.21](https://tip.golang.org/doc/go1.21#contextpkgcontext)
existe a opção de associar um `error` como causa do cancelamento de um contexto
e que pode der obtido utilizando a [função `context.Cause`](https://pkg.go.dev/context#Cause).

Caso o contexto tenha sido cancelado e exista uma causa _não-nula_, o valor retornado será o erro enviado como causa no momento do cancelamento. Já, se não existir uma causa específica, o valor será o mesmo da chamada `ctx.Err()`, que vimos anteriormente.

```go
if ctx.Err() != nil {
    // o contexto foi cancelado, só vamos retornar um erro informando o motivo, ok?
    return fmt.Errorf("failure before start long process: %w", context.Cause(ctx))
}
```

### Escutando o sinal de cancelamento — `ctx.Done`

Além da verificação ativa utilizando o `ctx.Err()`, também é possível **ouvir**
um sinal de cancelamento por meio de um canal do tipo `<-chan struct{}`,
retornado pela chamada `ctx.Done()`.

> [!INFO] Canais e structs vazias
> Os canais — do inglês _channels_ — são condutores de informação seguros em
> ambientes concorrentes. Isso significa que podem ser utilizados por diversas
> _goroutines_ ao mesmo tempo, sem riscos de condições de corrida.
>
> Você pode imaginar um canal como uma fila de mensagens. A operação `<-meucanal`
> recebe a próxima "mensagem", ou aguarda até que uma esteja disponível. Já
> `meucanal <- "conteúdo"` envia a mensagem `"conteúdo"` para o canal — e, caso
> a fila já esteja cheia, espera até que haja espaço disponível.
>
> Além disso, o operador `<-` pode aparecer antes ou depois na declaração de tipos
> para indicar se o canal é de **somente leitura** (`<-chan`) ou **somente escrita**
> (`chan<-`). No caso do `ctx.Done` temos um canal de **somente leitura** de
> struct vazia (`struct{}`)
>
> Para entender melhor como canais funcionam, recomendo visitar o  
> [Tour do Go — Concurrency](https://go.dev/tour/concurrency/2).
>
> Já uma `struct{}` é uma struct vazia — ela **não consome memória**.
> Por isso, o canal retornado por `ctx.Done()` é usado **apenas para sinalizar**
> o cancelamento, sem transmitir dados adicionais.
>
> Veja mais sobre structs vazias em: [The empty struct](https://dave.cheney.net/2014/03/25/the-empty-struct>)

Quando a chamada `<-ctx.Done()` é feita, o código aguarda o recebimento através
do canal, bloqueando a execução da _goroutine_ até receber algum conteúdo.

Pela natureza _bloqueante_ da chamada, geralmente usamos uma cláusula `select`
para escolher entre o resultado do `ctx.Done()` e algum outro canal, como no
exemplo abaixo:

[Exemplo completo no Go Playground](https://go.dev/play/p/Cz1H7pOg_HH)

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Goroutine que aguarda o cancelamento do contexto
    go func() {
        <-ctx.Done() // aguardando o recebimento da mensagem de cancelamento
        fmt.Println("contexto cancelado após recebimento do canal!")
    }()

    fmt.Println("fazendo algo importante...")
    time.Sleep(1 * time.Second)
    fmt.Println("cancelando o contexto")
    cancel()
    time.Sleep(1 * time.Second) // aguardando callback ser executado
}
```

```text
fazendo algo importante...
cancelando o contexto
callback: contexto cancelado!
```

### Usando callbacks para tratar um contexto cancelado — `context.AfterFunc`

> [!WARNING] Disponível a partir do Go 1.21

A função `context.AfterFunc(ctx Context, f func())` recebe um context `ctx`, que
ao ser cancelado executa a função `f`. Dessa forma, a função `f` age como um
[_callback_](https://en.wikipedia.org/wiki/Callback_(computer_programming))
acionado no cancelamento, útil em tarefas paralelas que não precisam retornar um
erro — mas precisam fazer algum tratamento quando o contexto for cancelado.

Essa chamada retorna uma `func() bool`  que pode ser chamada para desfazer a
associação dela com o contexto `ctx`, fazendo com que ela não seja mais chamada caso o contexto seja cancelado. Ela irá retornar `true` .

[Exemplo completo no Go Playground](https://go.dev/play/p/Xc_49SbTBnM)

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Callback que será chamado quando o contexto for cancelado
    callback := func() {
        fmt.Println("callback: contexto cancelado!")
    }
    stop := context.AfterFunc(ctx, callback)
    defer stop()

    callback2 := func() {
        fmt.Println("callback2: não serei executado")
    }
    stop2 := context.AfterFunc(ctx, callback2)
    fmt.Println("Conseguimos cancelar o callback2?", stop2())
    fmt.Println("Conseguimos cancelar o callback2?", stop2(), "(já está cancelado)")

    cancel()

    time.Sleep(1 * time.Second) // aguardando callbacks serem executados
}
```

```text
Conseguimos cancelar o callback2? true
Conseguimos cancelar o callback2? false (já está cancelado)
callback: contexto cancelado!
```

## Referências e material adicional

Estamos chegando ao final e não poderia deixar aqui algumas sugestões de material para você consolidar e até se aprofundar sobre concorrência, contextos e _goroutines_. Todos eles estão em inglês, mas você pode recorrer ao Google Tradutor ou alguma IA.

- [Go Concurrency Patterns: Pipelines and Cancelation](https://go.dev/blog/pipelines) explica o padrão de cancelamento de tarefas utilizando canais.
- [Go Concurrency Patterns: Context](https://go.dev/blog/context) introduz as funcionalidades do pacote `context`.
- [Context and Struct](https://go.dev/blog/context-and-structs) esclarece porque não é uma boa ideia passar contextos dentro de uma `struct`.
- [Learn Go with tests: Contexts](https://quii.gitbook.io/learn-go-with-tests/go-fundamentals/context) é uma abordagem que introduz os contextos com uma abordagem prática usando TDD (Test Driven Development).
- [The Complete Guide to Context in Golang: Efficient Concurrency Management](https://medium.com/@jamal.kaksouri/the-complete-guide-to-context-in-golang-efficient-concurrency-management-43d722f6eaea) além de apresentar o tema, se aprofunda um pouco em alguns cenários como requesições HTTP e conexão com banco de dados.
- [Graceful Shutdown in Go: Practical Patterns](https://victoriametrics.com/blog/go-graceful-shutdown/index.html) apresenta padrões utilizando canais e contexts para encerrar o ciclo de vida de forma controlada.

## Conclusão

Talvez você já tenha lido a [documentação do pacote context](https://pkg.go.dev/context)
anteriormente e sentido dificuldades em entender de quando e onde os contextos
podem ou devem ser usados. Isso pode ter acontecido porque a documentação se
concentra em informar quais funcionalidades existem, mas não os casos de uso
prático em que elas se aplicam.

Com esse artigo, foquei em tentar introduzir o tema com alguns exemplos práticos
e analogias — e espero que isso tenha te ajudado a entender melhor sobre o tema.

Se você tiver dúvidas, sugestões ou apenas quiser trocar uma ideia, sinta-se à
vontade para entrar em contato através dos comentários ou pelos links abaixo:

- [Bluesky](https://bsky.app/profile/dneto.me)
- [Discord](https://discord.com/users/100316148863614976)
- [GitHub](https://github.com/dneto)
- [LinkedIn](https://www.linkedin.com/in/dem%C3%A9trio-menezes-neto-54704a137/)
- [X (antigo Twitter)](https://x.com/dneto__)

Estou sempre aberto a feedbacks e novas ideias! 🚀

Até uma próxima 👋!
