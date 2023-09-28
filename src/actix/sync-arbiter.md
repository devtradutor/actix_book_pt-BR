
# SyncArbiter

Quando você executa atores (actors) normalmente, existem vários atores sendo executados na
thread do Árbitro do Sistema (System Arbiter), usando seu loop de eventos. No entanto, para cargas de trabalho intensivas de CPU
ou cargas de trabalho altamente concorrentes, você pode desejar que um ator execute múltiplas
instâncias em paralelo.

Isso é o que um SyncArbiter proporciona - a capacidade de iniciar múltiplas instâncias de
um Ator em um grupo de threads do sistema operacional.

É importante observar que um SyncArbiter pode hospedar apenas um único tipo de Ator. Isso significa
que você precisa criar um SyncArbiter para cada tipo de Ator que deseja executar dessa
maneira.

## Criando um Ator Síncrono

Ao implementar seu Ator para ser executado em um SyncArbiter, é necessário alterar o Contexto do seu Ator de `Context` para `SyncContext`.

```rust
use actix::prelude::*;

struct MySyncActor;

impl Actor for MySyncActor {
    type Context = SyncContext<Self>;
}
```

## Iniciando o Sync Arbiter

Agora que definimos um Ator Síncrono, podemos executá-lo em um pool de threads criado pelo
nosso `SyncArbiter`. Somente podemos controlar o número de threads no momento da criação do SyncArbiter - não podemos adicionar ou remover threads posteriormente.

```rust
use actix::prelude::*;

struct MySyncActor;

impl Actor for MySyncActor {
    type Context = SyncContext<Self>;
}

let addr = SyncArbiter::start(2, || MySyncActor);
```

Podemos nos comunicar com o endereço (addr) da mesma maneira que fizemos com nossos Atuadores anteriores que iniciamos. Podemos enviar mensagens, receber futuros e resultados, e muito mais..

## Caixas de correio de Ator Síncrono

Os Atuadores Síncronos não têm limites de caixa de correio, mas você ainda deve usar `do_send`, `try_send` e `send` como de costume para lidar com outros possíveis erros ou comportamentos síncronos vs. assíncronos.