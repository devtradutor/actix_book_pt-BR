# Começando

Vamos criar e executar nossa primeira aplicação do Actix. Vamos criar um novo projeto Cargo
que depende do actix e, em seguida, execute o aplicativo.

Na seção anterior, já instalamos a versão do Rust necessária. Agora vamos criar novos projetos cargo.

## Ator de Ping

Vamos escrever nossa primeira aplicação Actix! Comece criando um novo projeto cargo baseado em binário e navegue para o novo diretório:

```bash
cargo new actor-ping
cd actor-ping
```

Agora, adicione o actix como uma dependência do seu projeto, certificando-se de que o seu Cargo.toml contenha o seguinte:

```toml
[dependencies]
actix = "0.11.0"
actix-rt = "2.2" # <-- Runtime do actix
```

Vamos criar um ator(actor) que irá receber uma mensagem `Ping` e responder com o número de pings processados.

Um ator é um tipo que implementa a trait `Actor`:

```rust
use actix::prelude::*;

struct MyActor {
    count: usize,
}

impl Actor for MyActor {
    type Context = Context<Self>;
}
```

Cada ator possui um contexto de execução, para `MyActor` vamos usar `Context<A>`. Mais informações sobre os contextos dos atores estão disponíveis na próxima seção.

Agora precisamos definir a `Message` que o ator precisa aceitar. A mensagem pode ser qualquer tipo que implemente a trait `Message`.

```rust
use actix::prelude::*;

#[derive(Message)]
#[rtype(result = "usize")]
struct Ping(usize);
```

O objetivo principal da trait `Message` é definir um tipo de resultado. A mensagem `Ping` define `usize`, o que indica que qualquer ator que possa aceitar uma mensagem `Ping` precisa retornar um valor `usize`.

E, finalmente, precisamos declarar que nosso ator `MyActor` pode aceitar `Ping` e lidar com isso. Para fazer isso, o ator precisa implementar a trait `Handler<Ping>`.

```rust
impl Handler<Ping> for MyActor {
    type Result = usize;

    fn handle(&mut self, msg: Ping, _ctx: &mut Context<Self>) -> Self::Result {
        self.count += msg.0;

        self.count
    }
}
```

É isso. Agora só precisamos iniciar nosso ator e enviar uma mensagem para ele. O procedimento de inicialização depende da implementação do contexto do ator. No nosso caso, podemos usar `Context<A>`, que é baseado em tokio/future. Podemos iniciá-lo com `Actor::start()` ou `Actor::create()`. O primeiro é usado quando a instância do ator pode ser criada imediatamente. O segundo método é usado quando precisamos ter acesso ao objeto de contexto antes de criar a instância do ator. No caso do ator `MyActor`, podemos usar `start()`.

Toda comunicação com atores é feita por meio de um endereço. Você pode usar `do_send` para enviar uma mensagem sem aguardar uma resposta, 
ou `send` para enviar uma mensagem específica para um ator. Tanto `start()` quanto `create()` retornam um objeto de endereço.

No exemplo a seguir, vamos criar um ator `MyActor` e enviar uma mensagem.

Aqui estamos usando o `actix-rt` como forma de iniciar nosso sistema e executar nosso Future principal, para que possamos facilmente utilizar `.await` para aguardar as mensagens enviadas ao ator.

```rust
#[actix_rt::main] 
async fn main() {
    // start new actor
    let addr = MyActor { count: 10 }.start();

    // send message and get future for result
    let res = addr.send(Ping(10)).await;

    // handle() returns tokio handle
    println!("RESULT: {}", res.unwrap() == 20);

    // stop system and exit
    System::current().stop();
}
```

`#[actix_rt::main]` inicia o sistema e bloqueia até que o Future seja resolvido.

O exemplo do Ping está disponível no diretório [examples](https://github.com/actix/actix/tree/master/actix/examples/) do repositório.