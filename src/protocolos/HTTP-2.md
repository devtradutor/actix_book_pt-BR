
O `actix-web` atualiza automaticamente as conexões para o *HTTP/2*, se possível.

# Negociação

Quando uma das funcionalidades `rustls` ou `openssl` está habilitada, o `HttpServer` fornece os métodos [bind_rustls][bindrustls] e [bind_openssl][bindopenssl], respectivamente.

```toml
[dependencies]
actix-web = { version = "4", features = ["openssl"] }
openssl = { version = "0.10", features = ["v110"] }
```

```rust
use actix_web::{web, App, HttpRequest, HttpServer, Responder};
use openssl::ssl::{SslAcceptor, SslFiletype, SslMethod};

async fn index(_req: HttpRequest) -> impl Responder {
    "Hello."
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // load TLS keys
    // to create a self-signed temporary cert for testing:
    // `openssl req -x509 -newkey rsa:4096 -nodes -keyout key.pem -out cert.pem -days 365 -subj '/CN=localhost'`
    let mut builder = SslAcceptor::mozilla_intermediate(SslMethod::tls()).unwrap();
    builder
        .set_private_key_file("key.pem", SslFiletype::PEM)
        .unwrap();
    builder.set_certificate_chain_file("cert.pem").unwrap();

    HttpServer::new(|| App::new().route("/", web::get().to(index)))
        .bind_openssl("127.0.0.1:8080", builder)?
        .run()
        .await
}
```

As atualizações para o HTTP/2 descritas no [RFC 7540 §3.2][rfcsection32] não são suportadas. O início do HTTP/2 com conhecimento prévio é suportado tanto para conexões de texto simples quanto TLS ([RFC 7540 §3.4][rfcsection34]) (quando utilizando os construtores de serviço de nível mais baixo `actix-http`).

> Confira [os exemplos de TLS][examples] para um exemplo concreto.


[rfcsection32]: https://httpwg.org/specs/rfc7540.html#rfc.section.3.2
[rfcsection34]: https://httpwg.org/specs/rfc7540.html#rfc.section.3.4
[bindrustls]: https://docs.rs/actix-web/4/actix_web/struct.HttpServer.html#method.bind_rustls
[bindopenssl]: https://docs.rs/actix-web/4/actix_web/struct.HttpServer.html#method.bind_openssl
[tlsalpn]: https://tools.ietf.org/html/rfc7301
[examples]: https://github.com/actix/examples/tree/master/https-tls