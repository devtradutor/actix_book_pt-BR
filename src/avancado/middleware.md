# Middleware

O sistema de middleware do Actix Web nos permite adicionar comportamentos adicionais ao processamento de requisições/respostas. O middleware pode interagir com o processo de uma requisição recebida, permitindo modificar as requisições e interromper o processamento para retornar uma resposta antecipada.

O middleware também pode interagir com o processamento de respostas.

Tipicamente, o middleware está envolvido nas seguintes ações:

- Pré-processar a requisição
- Pós-processar uma resposta
- Modificar o estado da aplicação
- Acessar serviços externos (redis, registro, sessões)

O middleware é registrado para cada `App`, `scope` ou `Resource` e é executado em ordem oposta ao registro. Em geral, um _middleware_ é um tipo que implementa o [_Service trait_][servicetrait] e [_Transform trait_][transformtrait]. Cada método nas traits tem uma implementação padrão. Cada método pode retornar um resultado imediatamente ou um objeto _futuro_.

A seguir, demonstra-se a criação de um middleware simples:

```rust
use std::future::{ready, Ready};

use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error,
};
use futures_util::future::LocalBoxFuture;

// Existem duas etapas no processamento de middlewares.
// 1. Inicialização do middleware, a fábrica de middleware é chamada 
// com o próximo serviço na cadeia como parâmetro.
// 2. O método de chamada do middleware é chamado com a requisição normal.
pub struct SayHi;

// A fábrica de Middleware que é uma trait chamada `Transform`
// `S` - O tipo do próximo serviço
// `B` - O tipo do corpo da resposta
impl<S, B> Transform<S, ServiceRequest> for SayHi
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = SayHiMiddleware<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(SayHiMiddleware { service }))
    }
}

pub struct SayHiMiddleware<S> {
    service: S,
}

impl<S, B> Service<ServiceRequest> for SayHiMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    forward_ready!(service);

    fn call(&self, req: ServiceRequest) -> Self::Future {
        println!("Oi do início. Você solicitou: {}", req.path());

        let fut = self.service.call(req);

        Box::pin(async move {
            let res = fut.await?;

            println!("Oi da resposta");
            Ok(res)
        })
    }
}
```

Alternativamente, para casos de uso simples, você pode usar [_wrap_fn_][wrap_fn] para criar middlewares pequenos e ad hoc:

```rust
use actix_web::{dev::Service as _, web, App};
use futures_util::future::FutureExt;

#[actix_web::main]
async fn main() {
    let app = App::new()
        .wrap_fn(|req, srv| {
            println!("Hi from start. You requested: {}", req.path());
            srv.call(req).map(|res| {
                println!("Hi from response");
                res
            })
        })
        .route(
            "/index.html",
            web::get().to(|| async { "Hello, middleware!" }),
        );
}
```

> O Actix Web fornece vários middlewares úteis, como _logging_, _user sessions_, _compress_, etc.

**Aviso: se você usar `wrap()` ou `wrap_fn()` várias vezes, a última ocorrência será executada primeiro.**

## Logging

O registro é implementado como um middleware. É comum registrar um middleware de registro como o primeiro middleware para a aplicação. O middleware de registro deve ser registrado para cada aplicação.

O middleware `Logger` usa a biblioteca padrão de registro para registrar informações. Você deve habilitar o registro para o pacote _actix_web_ para ver o log de acesso ([env_logger][envlogger] ou similar).

### Uso

Crie um middleware `Logger` com o `format` especificado. O `Logger` padrão pode ser criado com o método `default`, que usa o formato padrão:

```ignore
  %a %t "%r" %s %b "%{Referer}i" "%{User-Agent}i" %T
```

```rust
use actix_web::middleware::Logger;
use env_logger::Env;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    use actix_web::{App, HttpServer};

    env_logger::init_from_env(Env::default().default_filter_or("info"));

    HttpServer::new(|| {
        App::new()
            .wrap(Logger::default())
            .wrap(Logger::new("%a %{User-Agent}i"))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

A seguir, temos um exemplo do formato padrão de registro:

```
INFO:actix_web::middleware::logger: 127.0.0.1:59934 [02/Dec/2017:00:21:43 -0800] "GET / HTTP/1.1" 302 0 "-" "curl/7.54.0" 0.000397
INFO:actix_web::middleware::logger: 127.0.0.1:59947 [02/Dec/2017:00:22:40 -0800] "GET /index.html HTTP/1.1" 200 0 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.13; rv:57.0) Gecko/20100101 Firefox/57.0" 0.000646
```

### Formato

- `%%` O sinal de porcentagem
- `%a` Endereço IP remoto (endereço IP do proxy se estiver usando um proxy reverso)
- `%t` Hora em que a requisição começou a ser processada
- `%P` O ID do processo filho que atendeu à requisição
- `%r` Primeira linha da requisição
- `%s` Código de status da resposta
- `%b` Tamanho da resposta em bytes, incluindo os cabeçalhos HTTP
- `%T` Tempo decorrido para atender à requisição, em segundos, com fração decimal no formato .06f
- `%D` Tempo decorrido para atender à requisição, em milissegundos
- `%{FOO}i` request.headers['FOO']
- `%{FOO}o` response.headers['FOO']
- `%{FOO}e` os.environ['FOO']

## Cabeçalhos padrão

Para definir cabeçalhos de resposta padrão, o middleware `DefaultHeaders` pode ser usado. O middleware _DefaultHeaders_ não define o cabeçalho se os cabeçalhos da resposta já contiverem um cabeçalho especificado.

```rust
use actix_web::{http::Method, middleware, web, App, HttpResponse, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .wrap(middleware::DefaultHeaders::new().add(("X-Version", "0.2")))
            .service(
                web::resource("/test")
                    .route(web::get().to(HttpResponse::Ok))
                    .route(web::method(Method::HEAD).to(HttpResponse::MethodNotAllowed)),
            )
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

## Sessões de usuário

O Actix Web fornece uma solução geral para gerenciamento de sessões. O middleware [**actix-session**][actixsession] pode usar vários tipos de backend para armazenar dados de sessão.

> Por padrão, apenas o backend de sessão de cookie está implementado. Outras implementações de backend podem ser adicionadas.

O [**CookieSession**][cookiesession] usa cookies como armazenamento de sessão. `CookieSessionBackend` cria sessões que estão limitadas a armazenar menos de 4000 bytes de dados, pois o payload deve caber em um único cookie. Um erro interno do servidor é gerado se uma sessão contiver mais de 4000 bytes.

Um cookie pode ter uma política de segurança _signed_ (assinado) ou _private_ (privado). Cada um tem um respectivo construtor `CookieSession`.

Um cookie _signed_ pode ser visualizado, mas não modificado pelo cliente. Um cookie _private_ (privado) não pode ser visualizado nem modificado pelo cliente.

Os construtores recebem uma chave como argumento. Esta é a chave privada para a sessão de cookie - quando esse valor é alterado, todos os dados da sessão são perdidos.

Em geral, você cria um middleware `SessionStorage` e o inicializa com uma implementação de backend específica, como `CookieSession`. Para acessar os dados da sessão, você deve usar o extractor [`Session`][requestsession]. Este método retorna um objeto [_Session_][sessionobj], que nos permite obter ou definir dados da sessão.
> `actix_session::storage::CookieSessionStore` está disponível na funcionalidade "cookie-session" do crate.

```rust
use actix_session::{Session, SessionMiddleware, storage::CookieSessionStore};
use actix_web::{web, App, Error, HttpResponse, HttpServer, cookie::Key};

async fn index(session: Session) -> Result<HttpResponse, Error> {
    // Acessa os dados da seseção
    if let Some(count) = session.get::<i32>("counter")? {
        session.insert("counter", count + 1)?;
    } else {
        session.insert("counter", 1)?;
    }

    Ok(HttpResponse::Ok().body(format!(
        "A contagem está em  {:?}!",
        session.get::<i32>("counter")?.unwrap()
    )))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .wrap(
                // cria um cookie baseado em seção middleware
                SessionMiddleware::builder(CookieSessionStore::default(), Key::from(&[0; 64]))
                    .cookie_secure(false)
                    .build()
            )
            .service(web::resource("/").to(index))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

## Manipuladores de erros

O middleware `ErrorHandlers` nos permite fornecer manipuladores personalizados para respostas de erro.

Você pode usar o método `ErrorHandlers::handler()` para registrar um manipulador de erro personalizado para um código de status específico. Você pode modificar uma resposta existente ou criar uma completamente nova. O manipulador de erro pode retornar uma resposta imediatamente ou retornar um futuro que se resolve em uma resposta.

```rust
use actix_web::middleware::{ErrorHandlerResponse, ErrorHandlers};
use actix_web::{
    dev,
    http::{header, StatusCode},
    web, App, HttpResponse, HttpServer, Result,
};

fn add_error_header<B>(mut res: dev::ServiceResponse<B>) -> Result<ErrorHandlerResponse<B>> {
    res.response_mut().headers_mut().insert(
        header::CONTENT_TYPE,
        header::HeaderValue::from_static("Error"),
    );

    Ok(ErrorHandlerResponse::Response(res.map_into_left_body()))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .wrap(
                ErrorHandlers::new()
                    .handler(StatusCode::INTERNAL_SERVER_ERROR, add_error_header),
            )
            .service(web::resource("/").route(web::get().to(HttpResponse::InternalServerError)))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

[sessionobj]: https://docs.rs/actix-session/0.7/actix_session/struct.Session.html
[requestsession]: https://docs.rs/actix-session/0.7/actix_session/struct.Session.html
[cookiesession]: https://docs.rs/actix-session/0.7/actix_session/storage/struct.CookieSessionStore.html
[actixsession]: https://docs.rs/actix-session/0.7/actix_session/
[envlogger]: https://docs.rs/env_logger/*/env_logger/
[servicetrait]: https://docs.rs/actix-web/4/actix_web/dev/trait.Service.html
[transformtrait]: https://docs.rs/actix-web/4/actix_web/dev/trait.Transform.html
[wrap_fn]: https://docs.rs/actix-web/4/actix_web/struct.App.html#method.wrap_fn