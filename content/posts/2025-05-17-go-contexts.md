+++
date = '2025-05-16T23:36:38-03:00'
draft = false
title = 'Contextos no Go'
layout = 'post'
tags = ['go', 'programação']
author = 'Demétrio Neto'
+++

## Introdução

Uma das responsabilidades que possuo onde trabalho altualmente é ensinar e
orientar pessoas menos experientes e uma das principais dúvidas que recebo
relacionadas a Go é sobre **contextos**.

> [!QUOTE]
> Mas eu não aguento mais passar isso em praticamente toda chamada que faço.\
> Outras linguagens não têm isso, por que eu tenho que me preocupar?
>
> \- Pessoa desenvolvedora vendo contextos em todo lugar

Para entender a utilidade dos contextos, precisamos estar cientes de que a
concorrência é uma realidade comum no desenvolvimento atual. Novas tarefas podem
surgir das mais diversas formas: processos, threads, goroutines
(ou [green threads](https://en.wikipedia.org/wiki/Green_thread)).

Mas, além de iniciar essas tarefas, é fundamental ter controle sobre elas,  
especialmente para saber quando e como interrompê-las caso algo não saia como  
esperado, evitando que fiquem rodando sem necessidade e/ou causem problemas no  
sistema.

Para ilustrar isso, vou usar uma analogia:

> [!ANALOGY] Analogia
> Imagine que você vai almoçar em um restaurante. Ao se sentar, escolhe um prato
> do cardápio, chama o garçom e faz seu pedido.
>
> O garçom anota o pedido e o envia para a cozinha, onde os cozinheiros começam a prepará-lo.
>
> Agora, imagine que, no meio disso, você recebe uma ligação urgente e precisa
> sair imediatamente. Então, você cancela o pedido.
>
> O que acontece com o prato que estava sendo preparado? Isso depende do
> funcionamento do restaurante: ele pode descartar tudo o que já foi feito, ou
> tentar reaproveitar parte dos ingredientes que ainda não estragaram.

Assim como o restaurante precisou lidar com o pedido cancelado, em sistemas
concorrentes também existem situações em que precisamos interromper uma tarefa
em andamento.

No exemplo, ao perceber que precisava sair e não voltaria, você cancelou o
pedido com o garçom — que então avisou a cozinha.

Às vezes, uma sinalização simples pode ser suficiente. Mas, em outros casos,
como no restaurante, o cancelamento precisa ser propagado por diferentes partes
do sistema e pode ter consequências importantes.

## Tempo e ciclo de vida das tarefas

Além do exemplo do restaurante, podemos pensar em situações mais próximas da
realidade de um desenvolvedor: pode ser o download de um arquivo grande, uma
requisição para uma API REST, uma consulta ao banco de dados ou a execução de
qualquer tarefa que pode levar muito tempo.

Como você faria para cancelar algo, se fosse necessário? E como notificaria esse
cancelamento em efeito cascata?

Cada linguagem, biblioteca ou framework oferece sua própria solução para lidar
tanto com o ciclo de vida quanto com o tempo de vida de fluxos e operações. Os
desenvolvedores do Go enxergaram essa necessidade — e é aí que entra o pacote
`context`.

Mas antes de falar sobre contextos, vale lembrar: o Go foi criado com a
filosofia de que **ser explícito é melhor do que esconder complexidade por trás de
abstrações implícitas**.

>[!QUOTE]
> Explícito é melhor que ímplicito
>
> \- Item 2 do [Zen of Python](https://peps.python.org/pep-0020/)

Apesar das coisas nem sempre serem tão explícitas assim no Python, esse é um bom
conselho a ser seguido para qualquer linguagem. Inclusive, recomendo também a
leitura do [Zen of Go](https://dave.cheney.net/2020/02/23/the-zen-of-go)

## O pacote `context`

O pacote `context` oferece um padrão para tratar esses cenários em que um sinal
de cancelamento precisa ser propagado dentro dos limites do código,
possibilitando o encerramento de forma graciosa, ou _gracious shutdown_, do
fluxo.

### A interface `context.Context`

Vamos começar olhando a [documentação da interface `context.Context`](https://pkg.go.dev/context#Context). Na descrição do tipo, vemos:

> [!QUOTE] Citação
> _A Context carries a deadline, a cancellation signal, and other values across API boundaries._\
> _Context's methods may be called by multiple goroutines simultaneously._

> [!TRANSLATION] Tradução
> _Um contexto carrega um prazo, um sinal de cancelamento, e outros valores através dos limites da API._\
> _Os métodos do tipo Context podem ser chamados por múltiplas goroutines simultâneamente_

## Lidando com o cancelamento

>[!COMMENT] _Não, não estamos falando de redes sociais_

A biblioteca padrão oferece algumas formas de tratar contextos cancelados:

### Verificando se o contexto já foi cancelado

Você pode ativamente verificar se o contexto já foi cancelado, essa abordagem é
útil para impedir que o fluxo prossiga. A forma mais comum é realizar a
verificação ao início da função, mas em alguns momentos pode ser importante
verificar ao final, para garantir que o fluxo não irá continuar.

#### Usando `ctx.Err()`

Você pode simplesmente verificar se retorno de `ctx.Err()` é _não-nulo_. A
função retornara `nil` caso o contexto ainda não tenha sido cancelado e algum
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

#### Identificando a causa do cancelamento

> [!WARNING] Essa funcionalidade só está disponível a partir da versão 1.21 do Go

A partir do [Go 1.21](https://tip.golang.org/doc/go1.21#contextpkgcontext)
existe a opção de associar um `error` como causa do cancelamento de um contexto que conseguimos obter utilizando a [função `context.Cause`](https://pkg.go.dev/context#Cause).

Caso o contexto tenha sido cancelado e exista uma causa _não-nula_, o valor retornado será o erro enviado como causa no momento do cancelamento. Já, se não existir uma causa específica, o valor será o mesmo da chamada `ctx.Err()`, que vimos anteriormente.

```go
if ctx.Err() != nil {
    // o contexto foi cancelado, só vamos retornar um erro informando o motivo, ok?
    return fmt.Errorf("failure before start long process: %w", context.Cause(ctx))
}
```

### Escutando o sinal de cancelamento

Além da verificação ativa utilizando o `ctx.Err()`, é possível receber um
`channel` que informa o cancelamento através da chamada `ctx.Done()`.

Quando a chamada `<-ctx.Done()` é feita, o código aguarda o recebimento através
do canal, bloqueando a execução da _goroutine_ até receber algum conteúdo.

```go
<-ctx.Done()
```

Pela natureza _bloqueante_ da chamada, geralmente usamos uma cláusula `select`
para escolher entre o resultado do `ctx.Done()` e algum outro canal, como no
exemplo abaixo:

```go
response := make(chan string, 1)

go func(){
    response <- longProcess()
}

select{
    case r := <-response:
        return r
    case <-ctx.Done():
        return fmt.Errorf("falha no processamento: %w", context.Cause(ctx))
}
```

### Usando callbacks para tratar um contexto cancelado

> [!WARNING] Essa funcionalidade só está disponível a partir da versão 1.21 do Go

A função `context.AfterFunc(ctx Context, f func())` recebe um context `ctx`, que
ao ser cancelado executa a função `f`. Dessa forma, a função `f` é um
[_callback_](https://en.wikipedia.org/wiki/Callback_(computer_programming))
para quando um contexto for cancelado e pode ser útil quando o código em
questão irá executar em paralelo, não precisa retornar um erro, mas precisa
fazer algum tratamento mais complexo quando o contexto for cancelado.

```go
callback := func(){
    //Essa função será executada quando o contexto for cancelado.
}

stop := context.AfterFunc(ctx, callback)
defer stop()

// processamento

return nil

```

## Como cancelar contextos?

Vimos, na sessão anterior, as opções que o pacote `context` oferece para tratar
um cancelamento e, se você foi curioso ou curiosa e também deu uma olhada na própria
definição da interface, possivelmente notou que não existe um método `Cancel()`
ou algo parecido, ou seja, um contexto não tem capacidade de ativar um sinal
de cancelamento para ele mesmo.

### Porque os contextos não devem cancelar a si próprios

Para explicar o motivo, vamos voltar ao exemplo do restaurante: imagine que o garçom
ou outro cliente pudesse cancelar seu pedido, ou pior, todos os pedidos. Você
também pode pensar em uma tarefa que inicia várias _goroutines_ em paralelo para
realizar o download de vários arquivos, a exposição de uma forma de cancelar
dentro do próprio contexto poderia possibilitar o cancelamento de outros
downloads iniciados pela mesma tarefa, seria uma tragédia 😟.

Logo, a existência da possibilidade de cancelar um contexto em qualquer escopo
seria terrivelmente perigoso e poderia impactar diversos fluxos de forma
imprevisível. Então, caso seja necessário ter um controle fino sobre o tempo de
vida de um determinado fluxo da sua aplicação, a recomendação é criar uma nova
instância do `context.Context` e o pacote `context` já oferece algumas formas
para criar contextos, vamos dar uma olhada nelas?

## Criando novos contextos

Para os exemplos a seguir, decidi seguir a _vibe_ de restaurante e criei
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

A outra, `kitchen`, representa a cozinha e tem o método `cook` representando o
ato de cozinhar o prato:

```go
type kitchen struct{}

func (k kitchen) cook(ctx context.Context, dish string) {
    fmt.Printf("cozinha: fingindo que estamos fazendo %q\n", dish)
    <-ctx.Done() // espera o contexto ser cancelado
    fmt.Printf("cozinha: parando de fazer %q: %s\n", dish, context.Cause(ctx))
}
```

Veja que estamos usando tanto o método `ctx.Done()`, como o `context.Cause(ctx)`
que já falamos anteriormente.

### Criando contextos com cancelamento

A função `context.WithCancel` possui dois retornos. O primeiro é o próprio
contexto, que deve ser repassado nas chamadas em que ele se faz necessário, já o
segundo é uma função de cancelamento que, ao ser chamada, irá cancelar o
contexto criado.

> [!WARNING]
> É sempre uma boa prática utilizar o `defer` nas funções de
> cancelamento, pois ajuda a evitar o vazamento de _goroutines_. [Exemplo completo no Go Playground](https://go.dev/play/p/xfY8hceWaHC)

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
`context.WithCancelCause`. [Exemplo completo no Go Playground](https://go.dev/play/p/rUI_qSkZTsF)

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

### Criando contextos com prazo de validade

Também existem opções para criar contextos com um prazo de
validade, ou seja, os contextos serão automaticamente cancelados após o período
informado. Existem duas funções para criar um contexto com prazo de validade:

#### `context.WithDeadline`

A `context.WithDeadline`, que recebe um `time.Time` e irá cancelar o
contexto após o tempo informado.

```go
deadline := time.Parse()
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()

// processamento 
```

#### `context.WithTimeout`

A `context.WithTimeout`, que recebe um `time.Duration` e irá cancelar o
contexto após o **periodo** informado.

### Criando contexto que carregam valores

Também existe a função `context.WithValue`, que permite a criação de um contexto
com valores armazenados internamente. Recomendo **bastante cautela** ao utilizar
contextos dessa forma, embora seja muito útil para passar agentes de métricas e
tracing ou dados de um request, como um request id, entre as diferentes camadas,
o abuso dessa opção pode causar problemas de clareza no código.

### Crie o seu próprio

Por se tratar de uma interface, você pode criar sua própria implementação.
Pessoalmente não recomendo seguir por esse caminho, pois nesses meus quase 10
anos de Go eu ainda não vi nenhum cenário que as interfaces fornecidas pela
biblioteca padrão não foram suficientes.

## Conclusão

Talvez você já tenha lido a [documentação do pacote context](https://pkg.go.dev/context)
anteriormente e sentido dificuldades em entender de quando e onde os contextos
podem ou devem ser usados. Isso pode ter acontecido porque a documentação se
preocupa em informar quais funcionalidades existem e não os casos de uso em que ela são
aplicáveis. Espero que os exemplos usados nesse artigo tenham te ajudado a entender melhor esse tal de parâmetro `ctx` que as funções eventualmente precisam.

Não deixe de conferir o link com referências e material adicional mais abaixo.

Até uma próxima 👋!

## Referências e material adicional

- [Go Concurrency Patterns: Pipelines and Cancelation](https://go.dev/blog/pipelines)
- [Go Concurrency Patterns: Context](https://go.dev/blog/context)
- [Context and Struct](https://go.dev/blog/context-and-structs)
- [Learn Go with tests: Contexts](https://quii.gitbook.io/learn-go-with-tests/go-fundamentals/context)
- [The Complete Guide to Context in Golang: Efficient Concurrency Management](https://medium.com/@jamal.kaksouri/the-complete-guide-to-context-in-golang-efficient-concurrency-management-43d722f6eaea)
- [Graceful Shutdown in Go: Practical Patterns](https://victoriametrics.com/blog/go-graceful-shutdown/index.html)
