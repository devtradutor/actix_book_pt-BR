# Extração de informações com segurança de tipo

O Actix Web oferece uma facilidade para acessar informações de requisição com segurança de tipo, chamada de _extractors_ (ou seja, `impl FromRequest`). Existem muitas implementações de extractors embutidas (veja os [implementadores](https://docs.rs/actix-web/latest/actix_web/trait.FromRequest.html#implementors)).

Um extractor pode ser acessado como argumento de uma função de handler. O Actix Web suporta até 12 extractors por função de handler. A posição do argumento não importa.

```rust
async fn index(path: web::Path<(String, String)>, json: web::Json<MyInfo>) -> impl Responder {
    let path = path.into_inner();
    format!("{} {} {} {}", path.0, path.1, json.id, json.username)
}
```
## Path

[_Path_][pathstruct] fornece informações extraídas do caminho da requisição. As partes do caminho que podem ser extraídas são chamadas de "segmentos dinâmicos" e são marcadas com chaves. Você pode desserializar qualquer segmento variável do caminho.

Por exemplo, para um recurso registrado no caminho `/users/{user_id}/{friend}`, dois segmentos podem ser desserializados: `user_id` e `friend`. Esses segmentos podem ser extraídos como uma tupla na ordem em que são declarados (por exemplo, `Path<(u32, String)>`).

```rust
use actix_web::{get, web, App, HttpServer, Result};

/// Extrair informações de caminho da URL "/users/{user_id}/{friend}".
/// {user_id} - Deserializa para um u32.
/// {friend} - Deserializa para uma String.
#[get("/users/{user_id}/{friend}")] // <- Define os parâmetros do caminho.
async fn index(path: web::Path<(u32, String)>) -> Result<String> {
    let (user_id, friend) = path.into_inner();
    Ok(format!("Bem-vindo {}, user_id {}!", friend, user_id))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(index))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

Também é possível extrair informações do caminho para um tipo que implementa o trait `Deserialize` do `serde`, combinando os nomes dos segmentos dinâmicos com os nomes dos campos. Aqui está um exemplo equivalente que usa o `serde` em vez de um tipo tupla.

```rust
use actix_web::{get, web, Result};
use serde::Deserialize;

#[derive(Deserialize)]
struct Info {
    user_id: u32,
    friend: String,
}

/// Extrair informações de caminho usando serde.
#[get("/users/{user_id}/{friend}")] // <- Define os parâmetros do caminho.
async fn index(info: web::Path<Info>) -> Result<String> {
    Ok(format!(
        "Bem-vindo {}, user_id {}!",
        info.friend, info.user_id
    ))
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

Como uma alternativa não segura de tipo, também é possível consultar (veja [documentação do `match_info`](https://docs.rs/actix-web/latest/actix_web/struct.HttpRequest.html#method.match_info)) a requisição para parâmetros de caminho por nome dentro de um handler:

```rust
#[get("/users/{user_id}/{friend}")] // <- Define os parâmetros do caminho.
async fn index(req: HttpRequest) -> Result<String> {
    let name: String = req.match_info().get("friend").unwrap().parse().unwrap();
    let userid: i32 = req.match_info().query("user_id").parse().unwrap();

    Ok(format!("Bem-vindo {}, user_id {}!", name, userid))
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

## Query

O tipo [`Query<T>`][querystruct] fornece funcionalidade de extração para os parâmetros de consulta da requisição. Por baixo, ele usa a crate `serde_urlencoded`.

```rust
use actix_web::{get, web, App, HttpServer};
use serde::Deserialize;

#[derive(Deserialize)]
struct Info {
    username: String,
}

// Este manipulador é chamado se a consulta for desserializada com sucesso em `Info`.
// Caso contrário, uma resposta de erro 400 Bad Request é retornada.
#[get("/")]
async fn index(info: web::Query<Info>) -> String {
    format!("Bem-vindo {}!", info.username)
}
```

## JSON

[`Json<T>`][jsonstruct] permite desserializar o corpo de uma requisição em uma estrutura. Para extrair informações digitadas do corpo de uma requisição, o tipo `T` deve implementar `serde::Deserialize`.

```rust
use actix_web::{post, web, App, HttpServer, Result};
use serde::Deserialize;

#[derive(Deserialize)]
struct Info {
    username: String,
}

/// desserializa `Info` do corpo da requisição
#[post("/submit")]
async fn submit(info: web::Json<Info>) -> Result<String> {
    Ok(format!("Welcome {}!", info.username))
}
```

Alguns extractors fornecem uma maneira de configurar o processo de extração. Para configurar um extractor, passe seu objeto de configuração para o método `.app_data()` do recurso. No caso do extractor _Json_, ele retorna um [_JsonConfig_][jsonconfig]. Você pode configurar o tamanho máximo da carga JSON, bem como uma função de tratamento de erros personalizada.

O exemplo a seguir limita o tamanho da carga útil para 4kb e usa um tratador de erros personalizado.

```rust
use actix_web::{error, web, App, HttpResponse, HttpServer, Responder};
use serde::Deserialize;

#[derive(Deserialize)]
struct Info {
    username: String,
}

/// desserializa `Info` do corpo da requisição, com tamanho máximo de carga útil de 4kb.
async fn index(info: web::Json<Info>) -> impl Responder {
    format!("Welcome {}!", info.username)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        let json_config = web::JsonConfig::default()
            .limit(4096)
            .error_handler(|err, _req| {
                // criar uma resposta de erro personalizada
                error::InternalError::from_response(err, HttpResponse::Conflict().finish())
                    .into()
            });

        App::new().service(
            web::resource("/")
                // alterar a configuração do extrator JSON
                .app_data(json_config)
                .route(web::post().to(index)),
        )
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

## Formulários Codificados em URL

Um corpo de formulário codificado em URL pode ser extraído para uma estrutura, assim como `Json<T>`. Esse tipo deve implementar `serde::Deserialize`.

[_FormConfig_][formconfig] permite configurar o processo de extração.

```rust
use actix_web::{post, web, App, HttpServer, Result};
use serde::Deserialize;

#[derive(Deserialize)]
struct FormData {
    username: String,
}

/// extrair dados de formulário usando serde
/// este manipulador é chamado apenas se o tipo de conteúdo for *x-www-form-urlencoded*
/// e o conteúdo da solicitação puder ser desserializado em uma estrutura `FormData`
#[post("/")]
async fn index(form: web::Form<FormData>) -> Result<String> {
    Ok(format!("Bem-vindo {}!", form.username))
}
```

## Outros

O Actix Web também oferece muitos outros extractors. Aqui estão alguns importantes:

- [`Data`][datastruct] - Para acessar partes do estado da aplicação.
- [`HttpRequest`][httprequest] - `HttpRequest` é ele mesmo um extractor, caso você precise de acesso a outras partes da requisição.
- `String` - Você pode converter a carga útil de uma requisição para uma `String`. [Um exemplo][stringexample] está disponível na documentação do Rust.
- [`Bytes`][bytes] - Você pode converter a carga útil de uma requisição em _Bytes_. [Um exemplo][bytesexample] está disponível na documentação do Rust.
- [`Payload`][payload] - Extractor de carga útil de baixo nível, principalmente para construir outros extractors. [Um exemplo][payloadexample] está disponível na documentação do Rust.

## Extractor de Estado da Aplicação

O estado da aplicação pode ser acessado a partir do manipulador com o extractor `web::Data`; no entanto, o estado é acessível apenas como uma referência somente leitura. Se você precisar de acesso mutável ao estado, ele deve ser implementado.

Aqui está um exemplo de um manipulador que armazena o número de requisições processadas:

```rust
use actix_web::{web, App, HttpServer, Responder};
use std::cell::Cell;

#[derive(Clone)]
struct AppState {
    count: Cell<usize>,
}

async fn show_count(data: web::Data<AppState>) -> impl Responder {
    format!("count: {}", data.count.get())
}

async fn add_one(data: web::Data<AppState>) -> impl Responder {
    let count = data.count.get();
    data.count.set(count + 1);

    format!("count: {}", data.count.get())
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let data = AppState {
        count: Cell::new(0),
    };

    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(data.clone()))
            .route("/", web::to(show_count))
            .route("/add", web::to(add_one))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

Embora esse manipulador funcione, `data.count` contará apenas o número de requisições tratadas _por cada thread de trabalho_. Para contar o número total de requisições em todas as threads, você deve usar um `Arc` compartilhado e [atomics][atomics].

```rust
use actix_web::{get, web, App, HttpServer, Responder};
use std::{
    cell::Cell,
    sync::atomic::{AtomicUsize, Ordering},
    sync::Arc,
};

#[derive(Clone)]
struct AppState {
    local_count: Cell<usize>,
    global_count: Arc<AtomicUsize>,
}

#[get("/")]
async fn show_count(data: web::Data<AppState>) -> impl Responder {
    format!(
        "global_count: {}\nlocal_count: {}",
        data.global_count.load(Ordering::Relaxed),
        data.local_count.get()
    )
}

#[get("/add")]
async fn add_one(data: web::Data<AppState>) -> impl Responder {
    data.global_count.fetch_add(1, Ordering::Relaxed);

    let local_count = data.local_count.get();
    data.local_count.set(local_count + 1);

    format!(
        "global_count: {}\nlocal_count: {}",
        data.global_count.load(Ordering::Relaxed),
        data.local_count.get()
    )
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let data = AppState {
        local_count: Cell::new(0),
        global_count: Arc::new(AtomicUsize::new(0)),
    };

    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(data.clone()))
            .service(show_count)
            .service(add_one)
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

**Observação**: Se você deseja compartilhar o estado _inteiro_ entre todas as threads, use `web::Data` e `app_data`, conforme descrito em [Estado Mutável Compartilhado][shared_mutable_state].

Tenha cuidado ao usar primitivas de sincronização bloqueadoras, como `Mutex` ou `RwLock`, dentro do estado da sua aplicação. O Actix Web trata as requisições de forma assíncrona. É um problema se a [_seção crítica_][critical_section] no seu handler for muito grande ou conter um ponto de `.await`. Se isso for uma preocupação, recomendamos que você também leia [o conselho do Tokio sobre o uso de `Mutex` bloqueador em código assíncrono][tokio_std_mutex].

[pathstruct]: https://docs.rs/actix-web/4/actix_web/dev/struct.Path.html
[querystruct]: https://docs.rs/actix-web/4/actix_web/web/struct.Query.html
[jsonstruct]: https://docs.rs/actix-web/4/actix_web/web/struct.Json.html
[jsonconfig]: https://docs.rs/actix-web/4/actix_web/web/struct.JsonConfig.html
[formconfig]: https://docs.rs/actix-web/4/actix_web/web/struct.FormConfig.html
[datastruct]: https://docs.rs/actix-web/4/actix_web/web/struct.Data.html
[httprequest]: https://docs.rs/actix-web/4/actix_web/struct.HttpRequest.html
[stringexample]: https://docs.rs/actix-web/4/actix_web/trait.FromRequest.html#impl-FromRequest-for-String
[bytes]: https://docs.rs/actix-web/4/actix_web/web/struct.Bytes.html
[bytesexample]: https://docs.rs/actix-web/4/actix_web/trait.FromRequest.html#impl-FromRequest-5
[payload]: https://docs.rs/actix-web/4/actix_web/web/struct.Payload.html
[payloadexample]: https://docs.rs/actix-web/4/actix_web/web/struct.Payload.html
[docsrs_match_info]: https://docs.rs/actix-web/latest/actix_web/struct.HttpRequest.html#method.match_info
[actix]: /actix/docs/
[atomics]: https://doc.rust-lang.org/std/sync/atomic/
[shared_mutable_state]: ./aplicacao.md#shared-mutable-state
[critical_section]: https://en.wikipedia.org/wiki/Critical_section
[tokio_std_mutex]: https://tokio.rs/tokio/tutorial/shared-state#on-using-stdsyncmutex