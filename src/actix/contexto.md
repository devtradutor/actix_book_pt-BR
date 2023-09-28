# Contexto

Os atores mantêm um contexto de execução interno, ou estado. Isso permite que um ator determine seu próprio endereço, altere os limites da caixa de mensagens ou interrompa sua execução.

## Caixa de mensagens(Mailbox)
Todas as mensagens vão primeiro para a caixa de mensagens do ator e, em seguida, o contexto de execução do ator chama os manipuladores de mensagem específicos. As caixas de mensagens em geral têm um limite de capacidade. A capacidade é específica para a implementação do contexto. Para o tipo `Context`, a capacidade é definida como 16 mensagens por padrão e pode ser aumentada com [`Context::set_mailbox_capacity()`].

```rust
struct MyActor;

impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        ctx.set_mailbox_capacity(1);
    }
}

let addr = MyActor.start();
```

Lembre-se de que isso não se aplica ao `Addr::do_send(M)`, que ignora o limite da fila da caixa de mensagens, ou ao `AsyncContext::notify(M)` e `AsyncContext::notify_later(M, Duration)`, que ignoram completamente a caixa de mensagens.

[`Context::set_mailbox_capacity()`]: https://docs.rs/actix/latest/actix/struct.Context.html#method.set_mailbox_capacity

## Obtendo o endereço dos atores

Um ator pode obter seu próprio endereço a partir de seu contexto. Talvez você queira reenfileirar um evento para mais tarde, ou queira transformar o tipo da mensagem. Talvez você queira responder com seu próprio endereço a uma mensagem. Se você quiser que um ator envie uma mensagem para si mesmo, dê uma olhada em `AsyncContext::notify(M)` em vez disso.

Para obter seu endereço do contexto, você chama o [`Context::address()`]. Um exemplo é:
```rust
struct MyActor;

struct WhoAmI;

impl Message for WhoAmI {
    type Result = Result<actix::Addr<MyActor>, ()>;
}

impl Actor for MyActor {
    type Context = Context<Self>;
}

impl Handler<WhoAmI> for MyActor {
    type Result = Result<actix::Addr<MyActor>, ()>;

    fn handle(&mut self, msg: WhoAmI, ctx: &mut Context<Self>) -> Self::Result {
        Ok(ctx.address())
    }
}

let who_addr = addr.do_send(WhoAmI{});
```

[`Context::address()`]: https://docs.rs/actix/latest/actix/struct.Context.html#method.address

## Parando um ator


Dentro do contexto de execução do ator, você pode escolher parar o processamento de futuras mensagens da caixa de correio do ator. Isso pode ser feito em resposta a uma condição de erro ou como parte do encerramento do programa. Para fazer isso, você chama [Context::stop()].

Este é um exemplo ajustado do Ping que para após receber 4 pings.

```rust
impl Handler<Ping> for MyActor {
    type Result = usize;

    fn handle(&mut self, msg: Ping, ctx: &mut Context<Self>) -> Self::Result {
        self.count += msg.0;

        if self.count > 5 {
            println!("Shutting down ping receiver.");
            ctx.stop()
        }

        self.count
    }
}

#[actix_rt::main]
async fn main() {
    // start new actor
    let addr = MyActor { count: 10 }.start();

    // send message and get future for result
    let addr_2 = addr.clone();
    let res = addr.send(Ping(6)).await;

    match res {
        Ok(_) => assert!(addr_2.try_send(Ping(6)).is_err()),
        _ => {}
    }
}
```

[`Context::stop()`]: https://docs.rs/actix/latest/actix/struct.Context.html#method.stop