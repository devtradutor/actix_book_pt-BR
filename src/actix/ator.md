# Ator (Actor)

O Actix é uma biblioteca em Rust que fornece um framework para o desenvolvimento de aplicativos concorrentes.

O Actix é construído no [Modelo de Ator(Actor Model)][Actor Model], que permite que os aplicativos sejam escritos como um grupo de "Atores" independentes, mas que cooperam entre si, se comunicando por meio de mensagens. Atores são objetos que encapsulam estado e comportamento e são executados dentro do *Sistema de Ator* fornecido pela biblioteca Actix.

Os atores são executados dentro de um contexto de execução específico [`Context<A>`]. O objeto de contexto está disponível apenas durante a execução. Cada ator tem um contexto de execução separado. O contexto de execução também controla o ciclo de vida de um ator.

Os atores se comunicam exclusivamente trocando mensagens. O ator remetente pode opcionalmente aguardar a resposta. Os atores não são referenciados diretamente, mas por meio de endereços.

Qualquer tipo Rust pode ser um ator, desde que implemente o traço [`Actor`].

Para ser capaz de lidar com uma mensagem específica, o ator deve fornecer uma implementação [`Handler<M>`] para essa mensagem. Todas as mensagens são tipadas estaticamente. A mensagem pode ser tratada de forma assíncrona. O ator pode criar outros atores ou adicionar futuros ou streams ao contexto de execução. A trait `Actor` fornece vários métodos que permitem controlar o ciclo de vida do ator.

[Actor Model]: https://en.wikipedia.org/wiki/Actor_model
[`Context<A>`]: ./context
[`Actor`]: https://docs.rs/actix/latest/actix/trait.Actor.html
[`Handler<M>`]: https://docs.rs/actix/latest/actix/trait.Handler.html

## Ciclo de vida do ator

### Iniciado

Um ator sempre começa no estado `Started`. Durante este estado, o método `started()` do ator é chamado. A trait `Actor` fornece uma implementação padrão para este método. O contexto do ator está disponível durante este estado e o ator pode iniciar outros atores, registrar fluxos assíncronos ou realizar qualquer outra configuração necessária.

### Executando

Após a chamada do método `started()` do ator, o ator transita para o estado `Running(Executando)`. O ator pode permanecer no estado `running` indefinidamente.

### Parando

O estado de execução do ator muda para o estado `stopping` nas seguintes situações:

* `Context::stop` é chamado pelo próprio ator
* todos os endereços para o ator são descartados, ou seja, nenhum outro ator faz referência a isso.
* nenhum objeto de evento está registrado no contexto.

Um ator pode restaurar do estado `stopping` para o estado `running` criando um novo endereço ou adicionando um objeto de evento, e retornando `Running::Continue`.

Se um ator mudou de estado para `stopping` porque `Context::stop()` foi chamado, então o contexto imediatamente para de processar as mensagens recebidas e chama `Actor::stopping()`. Se o ator não retornar ao estado `running`, todas as mensagens não processadas são descartadas.

Por padrão, este método retorna `Running::Stop`, o que confirma a operação de parada.

### Parado

Se o ator não modificar o contexto de execução durante o estado de parada, o estado do ator muda para `Stopped`. Este estado é considerado final e, neste ponto, o ator é descartado.

## Mensagem

Um ator se comunica com outros atores enviando mensagens. Em actix, todas as mensagens são tipadas. Uma mensagem pode ser qualquer tipo Rust que implemente a trait [`Message`]. `Message::Result` define o tipo de retorno. Vamos definir uma mensagem simples `Ping` - um ator que aceitará essa mensagem precisa retornar `Result<bool, std::io::Error>`.

```rust
use actix::prelude::*;

struct Ping;

impl Message for Ping {
    type Result = Result<bool, std::io::Error>;
}
```

[`Message`]: https://docs.rs/actix/latest/actix/trait.Message.html

## Gerando um ator

Como iniciar um ator depende do seu contexto. A criação de um novo ator assíncrono é feita por meio dos métodos `start` e `create` da trait [`Actor`]. Ele fornece várias maneiras diferentes de criar atores; para obter mais detalhes, consulte a documentação.

## Exemplo completo

```rust
use actix::prelude::*;

/// Define message
#[derive(Message)]
#[rtype(result = "Result<bool, std::io::Error>")]
struct Ping;

// Define actor
struct MyActor;

// Provide Actor implementation for our actor
impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Context<Self>) {
       println!("Actor is alive");
    }

    fn stopped(&mut self, ctx: &mut Context<Self>) {
       println!("Actor is stopped");
    }
}

/// Define handler for `Ping` message
impl Handler<Ping> for MyActor {
    type Result = Result<bool, std::io::Error>;

    fn handle(&mut self, msg: Ping, ctx: &mut Context<Self>) -> Self::Result {
        println!("Ping received");

        Ok(true)
    }
}

#[actix_rt::main]
async fn main() {
    // Start MyActor in current thread
    let addr = MyActor.start();

    // Send Ping message.
    // send() message returns Future object, that resolves to message result
    let result = addr.send(Ping).await;

    match result {
        Ok(res) => println!("Got result: {}", res.unwrap()),
        Err(err) => println!("Got error: {}", err),
    }
}
```

## Respondendo com um MessageResponse

Vamos dar uma olhada no tipo `Result` definido para a implementação de `impl Handler` no exemplo acima.
Veja como estamos retornando um `Result<bool, std::io::Error>`? Podemos responder à mensagem recebida pelo nosso ator com esse tipo porque ele implementa a trait `MessageResponse` para esse tipo. Aqui está a definição desta trait:

```rust
pub trait MessageResponse<A: Actor, M: Message> {
    fn handle(self, ctx: &mut A::Context, tx: Option<OneshotSender<M::Result>>);
}
```

Às vezes, faz sentido responder a mensagens recebidas com tipos que não têm essa trait implementada para eles. Quando isso acontece, podemos implementar o trait nós mesmos. Aqui está um exemplo em que estamos respondendo a uma mensagem `Ping` com uma `GotPing` e respondendo com `GotPong` para uma mensagem `Pong`.

```rust
use actix::dev::{MessageResponse, OneshotSender};
use actix::prelude::*;

#[derive(Message)]
#[rtype(result = "Responses")]
enum Messages {
    Ping,
    Pong,
}

enum Responses {
    GotPing,
    GotPong,
}

impl<A, M> MessageResponse<A, M> for Responses
where
    A: Actor,
    M: Message<Result = Responses>,
{
    fn handle(self, ctx: &mut A::Context, tx: Option<OneshotSender<M::Result>>) {
        if let Some(tx) = tx {
            tx.send(self);
        }
    }
}

// Define um ator
struct MyActor;

// Fornecer implementação de ator para nosso ator
impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, _ctx: &mut Context<Self>) {
        println!("Actor is alive");
    }

    fn stopped(&mut self, _ctx: &mut Context<Self>) {
        println!("Actor is stopped");
    }
}

/// Defina o manipulador para a enum `Messages`
impl Handler<Messages> for MyActor {
    type Result = Responses;

    fn handle(&mut self, msg: Messages, _ctx: &mut Context<Self>) -> Self::Result {
        match msg {
            Messages::Ping => Responses::GotPing,
            Messages::Pong => Responses::GotPong,
        }
    }
}

#[actix_rt::main]
async fn main() {
    // Start MyActor in current thread
    let addr = MyActor.start();

    // Send Ping message.
    // send() message returns Future object, that resolves to message result
    let ping_future = addr.send(Messages::Ping).await;
    let pong_future = addr.send(Messages::Pong).await;

    match pong_future {
        Ok(res) => match res {
            Responses::GotPing => println!("Ping received"),
            Responses::GotPong => println!("Pong received"),
        },
        Err(e) => println!("Actor is probably dead: {}", e),
    }

    match ping_future {
        Ok(res) => match res {
            Responses::GotPing => println!("Ping received"),
            Responses::GotPong => println!("Pong received"),
        },
        Err(e) => println!("Actor is probably dead: {}", e),
    }
}
```