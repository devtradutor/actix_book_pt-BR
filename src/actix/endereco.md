# Endereço

Os atores se comunicam exclusivamente trocando mensagens. O ator de envio pode opcionalmente esperar pela resposta. Os atores não podem ser referenciados diretamente, apenas por meio de seus endereços.

Existem várias maneiras de obter o endereço de um ator. A trait `Actor` fornece dois métodos auxiliares para iniciar um ator. Ambos retornam o endereço do ator iniciado.

Aqui está um exemplo de uso do método `Actor::start()`. Neste exemplo, o ator `MyActor` é assíncrono e é iniciado na mesma thread do chamador - tópicos são abordados no capítulo [SyncArbiter].

```rust
struct MyActor;
impl Actor for MyActor {
    type Context = Context<Self>;
}

let addr = MyActor.start();
```
Um ator assíncrono pode obter seu endereço a partir da struct `Context`. O contexto precisa implementar a trait `AsyncContext`. `AsyncContext::address()` fornece o endereço do ator.

```rust
struct MyActor;

impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Context<Self>) {
       let addr = ctx.address();
    }
}
```

[SyncArbiter]: ./sync-arbiter.md

## Mensagem

Para poder lidar com uma mensagem específica, o ator precisa fornecer uma implementação de [`Handler<M>`] para essa mensagem. Todas as mensagens têm um tipo estático. A mensagem pode ser tratada de forma assíncrona. O ator pode criar outros atores, adicionar futuros ou fluxos ao contexto de execução. A trait do ator fornece vários métodos que permitem controlar o ciclo de vida do ator.

Para enviar uma mensagem a um ator, é necessário usar o objeto `Addr`. O `Addr` fornece várias maneiras de enviar uma mensagem.

* `Addr::do_send(M)` - Este método ignora quaisquer erros no envio da mensagem. Se a caixa de correio estiver cheia, a mensagem ainda será enfileirada, ignorando o limite. 
Se a caixa de correio do ator estiver fechada, a mensagem será descartada silenciosamente. 
Este método não retorna o resultado, portanto, se a caixa de correio estiver fechada e ocorrer uma falha, você não terá uma indicação disso.

* `Addr::try_send(M)` - Este método tenta enviar a mensagem imediatamente. Se a caixa de correio estiver cheia ou fechada (ator está morto), este método retorna um [`SendError`].
* `Addr::send(M)` - Este método retorna um objeto futuro que se resolve para um resultado do processo de tratamento da mensagem. Se o objeto `Future` retornado for descartado, a mensagem será cancelada.

[`Handler<M>`]: https://docs.rs/actix/latest/actix/trait.Handler.html
[`SendError`]: https://docs.rs/actix/latest/actix/prelude/enum.SendError.html

## Destinatário(Recipient)
O `Recipient` é uma versão especializada de um endereço que suporta apenas um tipo de mensagem. Ele pode ser usado quando a mensagem precisa ser enviada para um tipo diferente de ator. Um objeto `Recipient` pode ser criado a partir de um endereço com o método `Addr::recipient()`.

Objetos de endereço requerem um tipo de ator, mas se quisermos enviar uma mensagem específica para um ator que possa lidar com essa mensagem, podemos usar a interface `Recipient`.

Por exemplo, o destinatário pode ser usado em um sistema de assinaturas. No exemplo a seguir, o ator `OrderEvents` envia uma mensagem `OrderShipped` para todos os assinantes. Um assinante pode ser qualquer ator que implemente a trait `Handler<OrderShipped>`.

```rust
use actix::prelude::*;

#[derive(Message)]
#[rtype(result = "()")]
struct OrderShipped(usize);

#[derive(Message)]
#[rtype(result = "()")]
struct Ship(usize);

/// Subscribe to order shipped event.
#[derive(Message)]
#[rtype(result = "()")]
struct Subscribe(pub Recipient<OrderShipped>);

/// Actor that provides order shipped event subscriptions
struct OrderEvents {
    subscribers: Vec<Recipient<OrderShipped>>,
}

impl OrderEvents {
    fn new() -> Self {
        OrderEvents {
            subscribers: vec![]
        }
    }
}

impl Actor for OrderEvents {
    type Context = Context<Self>;
}

impl OrderEvents {
    /// Send event to all subscribers
    fn notify(&mut self, order_id: usize) {
        for subscr in &self.subscribers {
           subscr.do_send(OrderShipped(order_id));
        }
    }
}

/// Subscribe to shipment event
impl Handler<Subscribe> for OrderEvents {
    type Result = ();

    fn handle(&mut self, msg: Subscribe, _: &mut Self::Context) {
        self.subscribers.push(msg.0);
    }
}

/// Subscribe to ship message
impl Handler<Ship> for OrderEvents {
    type Result = ();
    fn handle(&mut self, msg: Ship, ctx: &mut Self::Context) -> Self::Result {
        self.notify(msg.0);
        System::current().stop();
    }
}

/// Email Subscriber 
struct EmailSubscriber;
impl Actor for EmailSubscriber {
    type Context = Context<Self>;
}

impl Handler<OrderShipped> for EmailSubscriber {
    type Result = ();
    fn handle(&mut self, msg: OrderShipped, _ctx: &mut Self::Context) -> Self::Result {
        println!("Email sent for order {}", msg.0)
    }
    
}
struct SmsSubscriber;
impl Actor for SmsSubscriber {
    type Context = Context<Self>;
}

impl Handler<OrderShipped> for SmsSubscriber {
    type Result = ();
    fn handle(&mut self, msg: OrderShipped, _ctx: &mut Self::Context) -> Self::Result {
        println!("SMS sent for order {}", msg.0)
    }
    
}

fn main() {
    let system = System::new("events");
    let email_subscriber = Subscribe(EmailSubscriber{}.start().recipient());
    let sms_subscriber = Subscribe(SmsSubscriber{}.start().recipient());
    let order_event = OrderEvents::new().start();
    order_event.do_send(email_subscriber);
    order_event.do_send(sms_subscriber);
    order_event.do_send(Ship(1));
    system.run();
}
```