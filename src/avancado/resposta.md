# Resposta

Um padrão semelhante ao de um construtor é usado para construir uma instância de `HttpResponse`. O `HttpResponse` fornece vários métodos que retornam uma instância de `HttpResponseBuilder`, que implementa vários métodos auxiliares para construir respostas.

> Consulte a [documentação][responsebuilder] para descrições de tipos.

Os métodos `body`, `finish` e `json` finalizam a criação da resposta e retornam uma instância construída de _HttpResponse_. Se esses métodos forem chamados várias vezes na mesma instância do construtor, o construtor gerará um erro.

```rust
use actix_web::{http::header::ContentType, HttpResponse};

async fn index() -> HttpResponse {
    HttpResponse::Ok()
        .content_type(ContentType::plaintext())
        .insert_header(("X-Hdr", "sample"))
        .body("data")
}
```

## Resposta JSON

O tipo `Json` permite responder com dados JSON bem formados: basta retornar um valor do tipo `Json<T>`, onde `T` é o tipo de uma estrutura a ser serializada em _JSON_. O tipo `T` deve implementar a trait `Serialize` do _serde_.

Para que o exemplo a seguir funcione, você precisa adicionar `serde` às suas dependências no arquivo `Cargo.toml`:

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
```

```rust
use actix_web::{get, web, Responder, Result};
use serde::Serialize;

#[derive(Serialize)]
struct MyObj {
    name: String,
}

#[get("/a/{name}")]
async fn index(name: web::Path<String>) -> Result<impl Responder> {
    let obj = MyObj {
        name: name.to_string(),
    };
    Ok(web::Json(obj))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    use actix_web::{App, HttpServer};

    HttpServer::new(|| App::new().service(index))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

Usar o tipo `Json` dessa forma, em vez de chamar o método `json` em um `HttpResponse`, torna imediatamente claro que a função retorna JSON e não qualquer outro tipo de resposta.

## Codificação de conteúdo

O Actix Web pode comprimir automaticamente cargas úteis com o [_Compress middleware(Compactar middleware)_][compressmidddleware]. Os seguintes codecs são suportados:

- Brotli
- Gzip
- Deflate
- Identity

O cabeçalho `Content-Encoding` de uma resposta tem o valor padrão `ContentEncoding::Auto`, que realiza uma negociação automática de compressão de conteúdo com base no cabeçalho `Accept-Encoding` da requisição.

```rust
use actix_web::{get, middleware, App, HttpResponse, HttpServer};

#[get("/")]
async fn index() -> HttpResponse {
    HttpResponse::Ok().body("data")
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .wrap(middleware::Compress::default())
            .service(index)
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

Desative explicitamente a compressão de conteúdo em um manipulador definindo `Content-Encoding` como um valor `Identity`:

```rust
use actix_web::{
    get, http::header::ContentEncoding, middleware, App, HttpResponse, HttpServer,
};

#[get("/")]
async fn index() -> HttpResponse {
    HttpResponse::Ok()
        // v- Desativa a compressão 
        .insert_header(ContentEncoding::Identity)
        .body("data")
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .wrap(middleware::Compress::default())
            .service(index)
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

Ao lidar com um corpo já comprimido (por exemplo, ao servir ativos pré-comprimidos), defina manualmente o cabeçalho `Content-Encoding` na resposta para ignorar o middleware:

```rust
use actix_web::{
    get, http::header::ContentEncoding, middleware, App, HttpResponse, HttpServer,
};

static HELLO_WORLD: &[u8] = &[
    0x1f, 0x8b, 0x08, 0x00, 0xa2, 0x30, 0x10, 0x5c, 0x00, 0x03, 0xcb, 0x48, 0xcd, 0xc9, 0xc9,
    0x57, 0x28, 0xcf, 0x2f, 0xca, 0x49, 0xe1, 0x02, 0x00, 0x2d, 0x3b, 0x08, 0xaf, 0x0c, 0x00,
    0x00, 0x00,
];

#[get("/")]
async fn index() -> HttpResponse {
    HttpResponse::Ok()
        .insert_header(ContentEncoding::Gzip)
        .body(HELLO_WORLD)
}
```

[responsebuilder]: https://docs.rs/actix-web/4/actix_web/struct.HttpResponseBuilder.html
[compressmidddleware]: https://docs.rs/actix-web/4/actix_web/middleware/struct.Compress.html