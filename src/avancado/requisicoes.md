# Requisição JSON

Existem várias opções para desserialização do corpo JSON.

A primeira opção é usar o extrator _Json_. Primeiro, você define uma função manipuladora que aceita `Json<T>` como parâmetro e, em seguida, usa o método `.to()` para registrar essa manipuladora. Também é possível aceitar um objeto JSON válido arbitrário usando `serde_json::Value` como tipo `T`.

Primeiro exemplo de `requisição JSON` depende do `serde`:

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
```

Segundo exemplo de `requisição JSON` depende do `serde`, `serde_json` e `futures`:

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1"
futures = "0.3"
```

Se você quiser adicionar um valor padrão para um campo, consulte a [documentação](https://serde.rs/attr-default.html) do `serde`.

```rust
use actix_web::{web, App, HttpServer, Result};
use serde::Deserialize;

#[derive(Deserialize)]
struct Info {
    username: String,
}

/// Extraindo Info usando serde
async fn index(info: web::Json<Info>) -> Result<String> {
    Ok(format!("Bem-vindo {}!", info.username))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().route("/", web::post().to(index)))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

Você também pode carregar manualmente a carga útil na memória e, em seguida, desserializá-la.

No exemplo a seguir, vamos desserializar uma estrutura _MyObj_. Primeiro, precisamos carregar o corpo da requisição e, em seguida, desserializar o JSON em um objeto.

```rust
use actix_web::{error, post, web, App, Error, HttpResponse};
use futures::StreamExt;
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct MyObj {
    name: String,
    number: i32,
}

const MAX_SIZE: usize = 262_144; // max payload size is 256k

#[post("/")]
async fn index_manual(mut payload: web::Payload) -> Result<HttpResponse, Error> {
    // payload is a stream of Bytes objects
    let mut body = web::BytesMut::new();
    while let Some(chunk) = payload.next().await {
        let chunk = chunk?;
        // limit max size of in-memory payload
        if (body.len() + chunk.len()) > MAX_SIZE {
            return Err(error::ErrorBadRequest("overflow"));
        }
        body.extend_from_slice(&chunk);
    }

    // body is loaded, now we can deserialize serde-json
    let obj = serde_json::from_slice::<MyObj>(&body)?;
    Ok(HttpResponse::Ok().json(obj)) // <- send response
}
```

> Um exemplo completo para ambas as opções está disponível no [diretório de exemplos][examples].

## Codificação de conteúdo

O Actix Web descompacta automaticamente as cargas úteis. Os seguintes codecs são suportados:

- Brotli
- Gzip
- Deflate
- Zstd

Se os cabeçalhos da requisição contiverem um cabeçalho `Content-Encoding`, a carga útil da requisição é descompactada de acordo com o valor do cabeçalho. Vários codecs não são suportados, ou seja, `Content-Encoding: br, gzip`.

## Codificação de transferência segmentada

O Actix decodifica automaticamente a codificação _chunked_. O extrator [`web::Payload`][payloadextractor] já contém o fluxo de bytes decodificado. Se a carga útil da requisição estiver compactada com um dos codecs de compressão suportados (br, gzip, deflate), o fluxo de bytes será descompactado.

## Corpo multipartes

O Actix Web fornece suporte a fluxo de multipartes com uma crate externa, [`actix-multipart`][multipartcrate].

> Um exemplo completo está disponível no [diretório de exemplos][multipartexample].

## Corpo codificado por URL

O Actix Web oferece suporte a corpos codificados como _application/x-www-form-urlencoded_ com o extrator [`web::Form`][formencoded], que é resolvido para a instância desserializada. O tipo da instância deve implementar o trait `Deserialize` do _serde_.

O futuro _UrlEncoded_ pode retornar um erro em vários casos:

- o tipo de conteúdo não é `application/x-www-form-urlencoded`
- a codificação de transferência é `chunked`.
- o tamanho do conteúdo é maior que 256k
- a carga útil termina com um erro.

```rust
use actix_web::{post, web, HttpResponse};
use serde::Deserialize;

#[derive(Deserialize)]
struct FormData {
    username: String,
}

#[post("/")]
async fn index(form: web::Form<FormData>) -> HttpResponse {
    HttpResponse::Ok().body(format!("username: {}", form.username))
}
```

## Requisição de streaming

_HttpRequest_ é um fluxo de objetos `Bytes`. Pode ser usado para ler a carga útil do corpo da requisição.

No exemplo a seguir, lemos e imprimimos a carga útil da requisição em pedaços:

```rust
use actix_web::{get, web, Error, HttpResponse};
use futures::StreamExt;

#[get("/")]
async fn index(mut body: web::Payload) -> Result<HttpResponse, Error> {
    let mut bytes = web::BytesMut::new();
    while let Some(item) = body.next().await {
        let item = item?;
        println!("Chunk: {:?}", &item);
        bytes.extend_from_slice(&item);
    }

    Ok(HttpResponse::Ok().finish())
}
```

[examples]: https://github.com/actix/examples/tree/master/json/json
[multipartstruct]: https://docs.rs/actix-multipart/0.2/actix_multipart/struct.Multipart.html
[fieldstruct]: https://docs.rs/actix-multipart/0.2/actix_multipart/struct.Field.html
[multipartexample]: https://github.com/actix/examples/tree/master/forms/multipart
[urlencoded]: https://docs.rs/actix-web/4/actix_web/dev/struct.UrlEncoded.html
[payloadextractor]: https://docs.rs/actix-web/4/actix_web/web/struct.Payload.html
[multipartcrate]: https://crates.io/crates/actix-multipart
[formencoded]: https://docs.rs/actix-web/4/actix_web/web/struct.Form.html