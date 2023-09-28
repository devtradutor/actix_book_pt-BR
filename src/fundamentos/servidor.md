# O Servidor HTTP

O tipo [**HttpServer**][httpserverstruct] é responsável por servir requisições HTTP.

`HttpServer` aceita uma fábrica de aplicativos como parâmetro, e a fábrica de aplicativos deve ter fronteiras `Send` + `Sync`. Mais sobre isso na seção de _multi-threading_.

Para iniciar o servidor da web, ele primeiro deve ser vinculado a um soquete de rede. Use [`HttpServer::bind()`][bindmethod] com uma tupla ou uma string que represente o endereço do soquete, como `("127.0.0.1", 8080)` ou `"0.0.0.0:8080"`. Isso falhará se o soquete estiver sendo usado por outro aplicativo.

Após o sucesso do `bind`, use [`HttpServer::run()`][httpserver_run] para obter uma instância de [`Server`][server]. O `Server` deve ser aguardado (`await`) ou iniciado (`spawn`) para começar a processar as requisições e continuará em execução até receber um sinal de desligamento (como, por padrão, um `ctrl-c`; [leia mais aqui](#graceful-shutdown)).

```rust
use actix_web::{web, App, HttpResponse, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().route("/", web::get().to(HttpResponse::Ok)))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

## Multi-Thread

`HttpServer` inicia automaticamente um número de _workers_ HTTP, por padrão esse número é igual ao número de CPUs físicas no sistema. Esse número pode ser substituído pelo método [`HttpServer::workers()`][workers].

```rust
use actix_web::{web, App, HttpResponse, HttpServer};

#[actix_web::main]
async fn main() {
    HttpServer::new(|| App::new().route("/", web::get().to(HttpResponse::Ok))).workers(4);
    // <- Start 4 workers
}
```

Uma vez criados, os _workers_ recebem cada um uma instância separada de uma _aplicação_ para lidar com as requisições. O estado do aplicativo não é compartilhado entre as threads, e os manipuladores estão livres para manipular sua cópia do estado sem preocupações de concorrência.

O estado do aplicativo não precisa ser `Send` ou `Sync`, mas as fábricas de aplicativos devem ser `Send` + `Sync`.

Para compartilhar o estado entre as threads dos _workers_, use `Arc`/`Data`. Deve-se tomar cuidado especial ao introduzir compartilhamento e sincronização. Em muitos casos, custos de desempenho são introduzidos inadvertidamente como resultado do bloqueio do estado compartilhado para modificações.

Em alguns casos, esses custos podem ser reduzidos usando estratégias de bloqueio mais eficientes, por exemplo, usando [travas de leitura/escrita](https://doc.rust-lang.org/std/sync/struct.RwLock.html) em vez de [mutexes](https://doc.rust-lang.org/std/sync/struct.Mutex.html) para obter bloqueio não exclusivo, mas as implementações mais performáticas tendem a ser aquelas em que nenhum bloqueio ocorre.

Como cada thread do _worker_ processa suas requisições sequencialmente, manipuladores que bloqueiam a thread atual farão com que o _worker_ atual pare de processar novas requisições:

```rust
fn my_handler() -> impl Responder {
    std::thread::sleep(Duration::from_secs(5)); // <-- Prática ruim! Fará com que a thread do worker atual fique parada!
    "resposta"
}
```

Por esse motivo, qualquer operação longa que não seja vinculada à CPU (por exemplo, I/O, operações de banco de dados, etc.) deve ser expressa como futures ou funções assíncronas. Manipuladores assíncronos são executados concorrentemente pelas threads dos _workers_ e, portanto, não bloqueiam a execução:

```rust
async fn my_handler() -> impl Responder {
    tokio::time::sleep(Duration::from_secs(5)).await; // <-- Ok. A thread do worker tratará outras requisições aqui
    "resposta"
}
```

A mesma limitação se aplica aos extratores também. Quando uma função manipuladora recebe um argumento que implementa `FromRequest` e essa implementação bloqueia a thread atual, a thread do worker será bloqueada ao executar a manipuladora. Deve-se dar atenção especial ao implementar extratores por esse motivo e eles também devem ser implementados de forma assíncrona quando necessário.

## TLS / HTTPS

O Actix Web suporta duas implementações de TLS prontas para uso: `rustls` e `openssl`.

A funcionalidade `rustls` é para integração com o `rustls` e `openssl` é para integração com o `openssl`.

```toml
[dependencies]
actix-web = { version = "4", features = ["openssl"] }
openssl = { version = "0.10" }
```

```rust
use actix_web::{get, App, HttpRequest, HttpServer, Responder};
use openssl::ssl::{SslAcceptor, SslFiletype, SslMethod};

#[get("/")]
async fn index(_req: HttpRequest) -> impl Responder {
    "Bem-vindo!"
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // Carregar chaves TLS
    // Para criar um certificado temporário autoassinado para testes:
    // `openssl req -x509 -newkey rsa:4096 -nodes -keyout key.pem -out cert.pem -days 365 -subj '/CN=localhost'`   
    let mut builder = SslAcceptor::mozilla_intermediate(SslMethod::tls()).unwrap();
    builder
        .set_private_key_file("key.pem", SslFiletype::PEM)
        .unwrap();
    builder.set_certificate_chain_file("cert.pem").unwrap();

    HttpServer::new(|| App::new().service(index))
        .bind_openssl("127.0.0.1:8080", builder)?
        .run()
        .await
}
```
Para criar os arquivos key.pem e cert.pem, use o seguinte comando. **Preencha com o seu próprio assunto**

```bash
$ openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
  -days 365 -sha256 -subj "/C=CN/ST=Fujian/L=Xiamen/O=TVlinux/OU=Org/CN=muro.lxd"
```

Para remover a senha, em seguida, copie nopass.pem para key.pem

```bash
$ openssl rsa -in key.pem -out nopass.pem
```

## Keep-Alive

O Actix Web mantém as conexões abertas para aguardar solicitações subsequentes.

> O comportamento da conexão _keep alive_ é definido pelas configurações do servidor.

- `Duration::from_secs(75)` ou `KeepAlive::Timeout(75)`: habilita o temporizador de _keep-alive_ de 75 segundos.
- `KeepAlive::Os`: usa o _keep-alive_ do sistema operacional.
- `None` ou `KeepAlive::Disabled`: desabilita o _keep-alive_.

```rust
use actix_web::{http::KeepAlive, HttpServer};
use std::time::Duration;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // Defina o tempo de keep-alive para 75 segundos.
    let _one = HttpServer::new(app).keep_alive(Duration::from_secs(75));

    // Utilize o keep-alive do sistema operacional (geralmente com um tempo bastante longo).
    let _two = HttpServer::new(app).keep_alive(KeepAlive::Os);

    // Desativar keep-alive
    let _three = HttpServer::new(app).keep_alive(None);

    Ok(())
}
```

Se a primeira opção acima for selecionada, então o _keep-alive_ é ativado para solicitações HTTP/1.1 se a resposta não o desabilitar explicitamente, por exemplo, definindo o [tipo de conexão][httpconnectiontype] como `Close` ou `Upgrade`. Fechar uma conexão à força pode ser feito com [o método `force_close()` em `HttpResponseBuilder`](https://docs.rs/actix-web/4/actix_web/struct.HttpResponseBuilder.html#method.force_close).

> O _keep-alive_ está **desativado** para HTTP/1.0 e está **ativado** para HTTP/1.1 e HTTP/2.0.

```rust
use actix_web::{http, HttpRequest, HttpResponse};

async fn index(_req: HttpRequest) -> HttpResponse {
    let mut resp = HttpResponse::Ok()
        .force_close() // <- Encerre a conexão na HttpResponseBuilder.
        .finish();

    // Alternativamente, encerre a conexão na struct HttpResponse.
    resp.head_mut().set_connection_type(http::ConnectionType::Close);

    resp
}
```

## Desligamento Gracioso {#graceful-shutdown}

`HttpServer` suporta desligamento gracioso. Após receber um sinal de parada, os _workers_ têm um tempo específico para concluir o atendimento das solicitações. Quaisquer _workers_ ainda ativos após o tempo limite serão encerrados forçadamente. Por padrão, o tempo limite de desligamento é definido como 30 segundos. É possível alterar esse parâmetro com o método [`HttpServer::shutdown_timeout()`][shutdowntimeout].

`HttpServer` lida com vários sinais do sistema operacional. O _CTRL-C_ está disponível em todos os sistemas operacionais, outros sinais estão disponíveis em sistemas Unix.

- _SIGINT_ - Encerra os _workers_ à força.
- _SIGTERM_ - Encerra os _workers_ de forma graciosa.
- _SIGQUIT_ - Encerra os _workers_ à força.

> É possível desativar o tratamento de sinais com o método [`HttpServer::disable_signals()`][disablesignals].

[server]: https://docs.rs/actix-web/4/actix_web/dev/struct.Server.html
[httpserverstruct]: https://docs.rs/actix-web/4/actix_web/struct.HttpServer.html
[bindmethod]: https://docs.rs/actix-web/4/actix_web/struct.HttpServer.html#method.bind
[httpserver_run]: https://docs.rs/actix-web/4/actix_web/struct.HttpServer.html#method.run
[bindopensslmethod]: https://docs.rs/actix-web/4/actix_web/struct.HttpServer.html#method.bind_openssl
[bindrusttls]: https://docs.rs/actix-web/4/actix_web/struct.HttpServer.html#method.bind_rustls
[workers]: https://docs.rs/actix-web/4/actix_web/struct.HttpServer.html#method.workers
[tlsalpn]: https://tools.ietf.org/html/rfc7301
[exampleopenssl]: https://github.com/actix/examples/tree/master/security/openssl
[shutdowntimeout]: https://docs.rs/actix-web/4/actix_web/struct.HttpServer.html#method.shutdown_timeout
[disablesignals]: https://docs.rs/actix-web/4/actix_web/struct.HttpServer.html#method.disable_signals
[httpconnectiontype]: https://docs.rs/actix-web/4/actix_web/http/enum.ConnectionType.html