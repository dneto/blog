+++
date = '2025-05-16T23:36:38-03:00'
draft = false
title = 'Contextos no Go'
layout = 'post'
tags = ['Go', 'Programação']
author = 'Demétrio Neto'
+++

## Introdução

Uma das responsabilidades que possuo onde trabalho altualmente é ensinar e
orientar pessoas menos experientes e uma das principais dúvidas que recebo
relacionadas a Go é sobre **contextos**.

> Mas eu não aguento mais passar isso em praticamente toda chamada que faço.\
> Outras linguagens não têm isso, por que eu tenho que me preocupar 🤯?
>
> _- Pessoa desenvolvedora começando a aprender Go_

Antes de começar a falar qualquer coisa sobre os contextos, é importante lembrar
que Go é uma linguagem criada com a filosofia de que deixar as coisas
_explícitas_ é melhor do que ter tudo _implícito_.

> Explícito é melhor que ímplicito
>
> _- Item 2 do [Zen of Python](https://peps.python.org/pep-0020/)_
>

_Apesar das coisas nem sempre serem tão explícitas assim no Python, esse é um bom conselho a ser seguido para qualquer linguagem. Inclusive, recomendo também a leitura do [Zen of Go](https://dave.cheney.net/2020/02/23/the-zen-of-go)_

Bom, se você já parou para ler documentação do pacote
[context](https://pkg.go.dev/context) alguma vez, talvez tenha sentido
dificuldade em entender de quando e onde se deve usar contextos. Isso porque a
documentação se preocupa em informar quais funcionalidades existem e não casos
de uso reais.

Nessa publicação, vou tentar explicar como os contextos podem ser úteis e porque
é importante que eles sejam passados ou repassados entre chamadas que podem
demorar.

## Interropendo fluxos

Imagine uma operação longa sendo realizada. Pode ser o download de um arquivo
grande, uma requisição para uma API REST, uma consulta ao banco de dados ou a
execução de tarefa que vai demorar muito tempo. Como você faria pra cancelar
algo, se fosse necessário?

Em outras linguagens, cada biblioteca ou framework oferece sua solução para
lidar com seus respectivos ciclos de vida. Os desenvolvedores do Go enxergaram a
necessidade de gerenciar o e resolveram implementaram o pacote `context` na
biblioteca padrão do Go. Convenientemente, esse padrão não só é útil para
requisições web, mas para qualquer tipo de tarefa que possa ser cancelada,
fazendo com que esse padrão seja amplamente utilizado nessa linguagem.

Já entendemos que existem situações que precisamos de um controle mais fino
sobre o ciclo de vida de um fluxo. Mas como fazer isso em Go?
O artigo [Go Concurrency Patterns: Pipelines and Cancelation](https://go.dev/blog/pipelines)
já mostra uma forma de cancelar um processamento em _goroutines_ usando canais
no estilo `done := make(chan struct{})`. "Outro" (vamos ver mais tarde o porque
dessas aspas) padrão nos é apresentada em outro artigo: [Go Concurrency Patterns:Context](https://go.dev/blog/context),
que nos apresenta o pacote [`context`](https://pkg.go.dev/context) que, olha só,
estamos falando nesse post.

Com esses padrões é possível sinalizar que um fluxo deve ser interrompido,
possibilitando o encerramento de forma _graciosa_ (_gracious shutdown_).

## A interface context.Context

Vamos dar uma olhada na documentação da interface [`context.Context`](https://pkg.go.dev/context#Context):

> _A Context carries a deadline, a cancellation signal, and other values across API boundaries._\
> _Context's methods may be called by multiple goroutines simultaneously._

Agora, em tradução livre para o português:

> _Um contexto carrega um prazo, um sinal de cancelamento, e outros valores através dos limites da API._\
> _Os métodos do tipo Context podem ser chamados por múltiplas goroutines simultâneamente_

Se você foi curioso e deu uma olhada na própria definição da interface,
possivelmente notou que ela não exige um método `Cancel()`. Ora, a existência da
possibilidade de cancelar um contexto em qualquer escopo seria terrivelmente
perigoso, e poderia impactar no ciclo de vida de muitos fluxos em paralelo de
forma imprevisível. Imagine uma tarefa que inicia várias _goroutines_ em
paralelo para realizar o download de vários arquivos, a exposição de uma forma
de cancelar dentro do próprio contexto poderia possibilitar o cancelamento de
outros downloads iniciados pela mesma tarefa, seria uma tragédia 😟.

Então, caso você deseje ter um controle fino sobre o ciclo de vida de um
determinado fluxo da sua aplicação, a recomendação é criar uma nova instância do
`context.Context`, o pacote `context` já oferece algumas formas para criar
contextos. Vamos dar uma olhada nelas?

## Criando novos contextos

Atualmente, existem três formas de criar um novo contexto utilizando o pacote
`context`, além da possibilidade de criar seu próprio contexto implementando a
interface `context.Context`.

### Com cancelamento

A função `context.WithCancel` possui dois retornos. O primeiro é o próprio
contexto, que deve ser repassado nas chamadas em que ele se faz necessário, já o
segundo é uma função de cancelamento que, ao ser chamada, irá cancelar o
contexto criado.

```go
func contextWithCancel() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // processamento
}
```

### Com deadline e timeout

O pacote `context` também oferece opções para criar contextos com uma data
limite. A função `context.WithDeadline` recebe um `time.Time` e irá cancelar o
contexto após o tempo informado.

Já a função `context.WithTimeout` recebe um `time.Duration` e irá cancelar o
contexto após o **periodo** informado.

### Com valores

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

## Lidando com o cancelamento

_Não, não estamos falando de redes sociais_ 🫠

A biblioteca padrão oferece três formas de tratar contextos cancelados:

1. Verificar o erro retornado pelo `context.Err()`
2. Escutar o canal `context.Done()`
3. Implementar um callback e usar com o `context.AfterFunc()`

### Verificando o context.Err()

Essa abordagem é útil se você quiser verificar se o contexto já foi cancelado
antes do processamento realmente acontecer, você pode simplesmente verificar se
`ctx.Err() != nil`, a função retornara `nil` caso o contexto ainda não tenha
sido cancelado.

```go
func longProcess(ctx context.Context) error {
    if ctx.Err() != nil {
        // o contexto foi cancelado, só vamos retornar um erro informando o motivo, ok? 👍
        return fmt.Errorf("failure before start long process: %w", ctx.Err())
    }

    realLongProcess(ctx)

    return nil 
}
```

### Escutando o ctx.Done()

A função `ctx.Done()` retorna um canal que informa quando o trabalho de um
determinado contexto deve ser interrompido.

```go
func longProcess(ctx context.Context) error {
    result := make(chan string, 1)

    // Aqui usamos o select para escolher entre o resultado do processamento e o
    // cancelamento do contexto, executando o código correspondente.
    for{
        select {
            case m := <-readMessage():
                // processa o resultado
            case <-ctx.Done():
                // tratamento para o cancelamento do contexto
                return fmt.Errorf("failure during long process: %w", ctx.Err())
        }
    }
}
```

### Usando o context.AfterFunc

```go
func longProcess(ctx context.Context) error {
    stop := context.AfterFunc(ctx, func(){
        // quando o contexto for cancelado, esse código será executado
    })
    defer stop()

    // processamento

    return nil
}
```

## Referências e material adicional

- [Go Concurrency Patterns: Pipelines and Cancelation](https://go.dev/blog/pipelines)
- [Go Concurrency Patterns: Context](https://go.dev/blog/context)
- [Context and Struct](https://go.dev/blog/context-and-structs)
- [Learn Go with tests: Contexts](https://quii.gitbook.io/learn-go-with-tests/go-fundamentals/context)
- [The Complete Guide to Context in Golang: Efficient Concurrency Management](https://medium.com/@jamal.kaksouri/the-complete-guide-to-context-in-golang-efficient-concurrency-management-43d722f6eaea)
- [Graceful Shutdown in Go: Practical Patterns](https://victoriametrics.com/blog/go-graceful-shutdown/index.html)

Espero que tenha ajudado você e até uma próxima 👋.
