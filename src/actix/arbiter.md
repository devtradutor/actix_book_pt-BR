# Arbiter

Os `Arbiters` fornecem um contexto de execução assíncrona para `Actors`, `funções` e `futures`. Onde um ator contém um `Context` que define seu estado de execução específico do ator, os `Arbiters` hospedam o ambiente onde um ator é executado.

Como resultado, os `Arbiters` desempenham várias funções. Mais notavelmente, eles são capazes de criar uma nova thread do sistema operacional, executar um loop de eventos, criar tarefas de forma assíncrona nesse loop de eventos e atuar como auxiliares para tarefas assíncronas.

## Sistema e Arbiter

Em todos os nossos exemplos de código anteriores, a função `System::new` cria um `Arbiter` para que seus atores possam ser executados dentro dele. Quando você chama `start()` no seu ator, ele está sendo executado dentro da thread do `Arbiter` do `System`. Em muitos casos, isso é tudo o que você precisa para um programa usando o Actix.

Embora ele use apenas uma thread, ele utiliza o padrão de loop de eventos muito eficiente, que funciona bem para eventos assíncronos. Para lidar com tarefas síncronas que consomem muito CPU, é melhor evitar bloquear o loop de eventos e, em vez disso, transferir o processamento para outras threads. Para esse caso de uso, leia a próxima seção e considere usar o [`SyncArbiter`](./sync-arbiter.md).

## O loop de eventos
Um `Arbiter` controla uma thread com um pool de eventos. Quando um `Arbiter` inicia uma tarefa (por meio de `Arbiter::spawn`, `Context<Actor>::run_later` ou construções similares), o `Arbiter` coloca a tarefa na fila de execução dessa fila de tarefas. Quando você pensa em `Arbiter`, pode pensar em um "loop de eventos de thread única".

O Actix em geral suporta a concorrência, mas os `Arbiters` normais (não os `SyncArbiters`) não suportam. Para usar o Actix de forma concorrente, você pode iniciar vários `Arbiters` usando `Arbiter::new`, `ArbiterBuilder` ou `Arbiter::start`.

Quando você cria um novo `Arbiter`, isso cria um novo contexto de execução para os `Actors`. A nova thread está disponível para adicionar novos `Actors` a ela, mas os `Actors` não podem se mover livremente entre os `Arbiters`: eles estão vinculados ao `Arbiter` em que foram iniciados. No entanto, `Actors` em diferentes `Arbiters` ainda podem se comunicar entre si usando os métodos normais de `Addr`/`Recipient`. O método de passagem de mensagens é agnóstico em relação a se os `Actors` estão sendo executados nos mesmos `Arbiters` ou em `Arbiters` diferentes.

## Usando o Arbiter para resolver eventos assíncronos

Se você não é um especialista em Rust Futures, o `Arbiter` pode ser uma ajuda e uma forma simples de lidar com a resolução de eventos assíncronos em ordem. Vamos considerar que temos dois atores, A e B, e queremos executar um evento em B somente após a conclusão de um resultado em A. Podemos usar `Arbiter::spawn` para ajudar nessa tarefa.

```rust
use actix::prelude::*;

struct SumActor {}

impl Actor for SumActor {
    type Context = Context<Self>;
}

#[derive(Message)]
#[rtype(result = "usize")]
struct Value(usize, usize);

impl Handler<Value> for SumActor {
    type Result = usize;

    fn handle(&mut self, msg: Value, _ctx: &mut Context<Self>) -> Self::Result {
        msg.0 + msg.1
    }
}

struct DisplayActor {}

impl Actor for DisplayActor {
    type Context = Context<Self>;
}

#[derive(Message)]
#[rtype(result = "()")]
struct Display(usize);

impl Handler<Display> for DisplayActor {
    type Result = ();

    fn handle(&mut self, msg: Display, _ctx: &mut Context<Self>) -> Self::Result {
        println!("Got {:?}", msg.0);
    }
}

fn main() {
    let system = System::new("single-arbiter-example");

    // Define an execution flow using futures
    let execution = async {
        // `Actor::start` spawns the `Actor` on the *current* `Arbiter`, which
        // in this case is the System arbiter
        let sum_addr = SumActor {}.start();
        let dis_addr = DisplayActor {}.start();

        // Start by sending a `Value(6, 7)` to our `SumActor`.
        // `Addr::send` responds with a `Request`, which implements `Future`.
        // When awaited, it will resolve to a `Result<usize, MailboxError>`.
        let sum_result = sum_addr.send(Value(6, 7)).await;

        match sum_result {
            Ok(res) => {
                // `res` is now the `usize` returned from `SumActor` as a response to `Value(6, 7)`
                // Once the future is complete, send the successful response (`usize`)
                // to the `DisplayActor` wrapped in a `Display
                dis_addr.send(Display(res)).await;
            }
            Err(e) => {
                eprintln!("Encountered mailbox error: {:?}", e);
            }
        };
    };

    // Spawn the future onto the current Arbiter/event loop
    Arbiter::spawn(execution);

    // We only want to do one computation in this example, so we
    // shut down the `System` which will stop any Arbiters within
    // it (including the System Arbiter), which will in turn stop
    // any Actor Contexts running within those Arbiters, finally
    // shutting down all Actors.
    System::current().stop();

    system.run();
}
```