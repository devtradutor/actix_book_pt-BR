## Escrevendo um Aplicativo

O `actix-web` fornece várias primitivas para construir servidores e aplicativos da web com Rust. Ele oferece roteamento, middleware, pré-processamento de solicitações, pós-processamento de respostas, etc.

Todos os servidores `actix-web` são construídos em torno da instância [`App`][app]. É usado para registrar rotas para recursos e middleware. Também armazena o estado do aplicativo compartilhado entre todos os manipuladores dentro do mesmo escopo.

O [`scope`(escopo)][scope] de um aplicativo atua como um espaço para todos os caminhos de rota, ou seja, todas as rotas para um escopo específico de aplicativo têm o mesmo prefixo de caminho de URL. O prefixo do aplicativo sempre contém uma barra "/" inicial. Se um prefixo fornecido não contiver uma barra inicial, ela será inserida automaticamente. O prefixo deve consistir em segmentos de caminho de valor.

> Para um aplicativo com escopo `/app`, qualquer solicitação com os caminhos `/app`, `/app/` ou `/app/test` corresponderia; no entanto, o caminho `/application` não corresponderia.

```rust
use actix_web::{web, App, HttpServer, Responder};

async fn index() -> impl Responder {
    "Olá Mundo!"
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new().service(
            //  prefixa todos os recursos e rotas anexados a ele...
            web::scope("/app")
                // ...portanto, isso trata solicitações para `GET /app/index.html`.
                .route("/index.html", web::get().to(index)),
        )
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

Neste exemplo, um aplicativo com o prefixo `/app` e um recurso `index.html` é criado. Este recurso está disponível através do URL `/app/index.html`.

> Para mais informações, consulte a seção [URL Dispatch][usingappprefix].

## Estado

O estado do aplicativo é compartilhado com todas as rotas e recursos dentro do mesmo escopo. O estado pode ser acessado usando o extrator [`web::Data<T>`][data], onde `T` é o tipo do estado. O estado também é acessível para o middleware.

Vamos escrever um aplicativo simples e armazenar o nome do aplicativo no estado:

```rust
use actix_web::{get, web, App, HttpServer};

// Está estrutura representa um estado
struct AppState {
    app_name: String,
}

#[get("/")]
async fn index(data: web::Data<AppState>) -> String {
    let app_name = &data.app_name; // <- obtem o app_name
    format!("Olá {app_name}!") // <- resposta com o  app_name
}
```

Em seguida, passe o estado ao inicializar o `App` e iniciar o aplicativo:

```rust
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .app_data(web::Data::new(AppState {
                app_name: String::from("Actix Web"),
            }))
            .service(index)
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

Qualquer número de tipos de estado pode ser registrado dentro do aplicativo.

## Estado Compartilhado Mutável {#shared-mutable-state}

O `HttpServer` aceita uma fábrica de aplicativos em vez de uma instância de aplicativo. Um `HttpServer` constrói uma instância de aplicativo para cada thread. Portanto, os dados do aplicativo devem ser construídos várias vezes. Se você deseja compartilhar dados entre diferentes threads, um objeto compartilhável deve ser usado, por exemplo, `Send` + `Sync`.

Internamente, [`web::Data`][data] usa `Arc`. Portanto, para evitar a criação de dois `Arc`s, devemos criar nossos dados antes de registrá-los usando [`App::app_data()`][appdata].

No exemplo a seguir, escreveremos um aplicativo com estado compartilhado mutável. Primeiro, definimos nosso estado e criamos nosso manipulador:

```rust
use actix_web::{web, App, HttpServer};
use std::sync::Mutex;

struct AppStateWithCounter {
    counter: Mutex<i32>, // <- Mutex é necessário para realizar mutações com segurança entre threads.
}

async fn index(data: web::Data<AppStateWithCounter>) -> String {
    let mut counter = data.counter.lock().unwrap(); // <- obter MutexGuard do contador
    *counter += 1; // <- acessa o counter dentro do  MutexGuard

    format!("Número da solicitação: {counter}") // <- resposta com counter
}
```

e registramos os dados em um `App`:

```rust
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // Observação: web::Data criado _fora_ do fechamento HttpServer::new
    let counter = web::Data::new(AppStateWithCounter {
        counter: Mutex::new(0),
    });

    HttpServer::new(move || {
        //mova o counter para o fechamento
        App::new()
            .app_data(counter.clone()) // <- registre os dados criados
            .route("/", web::get().to(index))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

Principais pontos a serem observados:
- O estado inicializado _dentro_ do fechamento passado para `HttpServer::new` é local para a thread do trabalhador e pode ficar dessincronizado se for modificado.
- Para obter um estado _globalmente compartilhado_, ele deve ser criado **fora**(outside) do fechamento passado para `HttpServer::new` e movido/clonado.

## Usando um Escopo de Aplicativo para Compor Aplicativos

O método [`web::scope()`][webscope] permite definir um prefixo de grupo de recursos. Esse escopo representa um prefixo de recurso que será adicionado a todos os padrões de recurso adicionados pela configuração de recursos. Isso pode ser usado para montar um conjunto de rotas em uma localização diferente da pretendida pelo autor original, mantendo os mesmos nomes de recursos.

Por exemplo:

```rust
#[actix_web::main]
async fn main() {
    let scope = web::scope("/users").service(show_users);
    App::new().service(scope);
}
```

No exemplo acima, a rota `show_users` terá um padrão de rota efetivo de `/users/show` em vez de `/show`, porque o argumento de escopo do aplicativo será prefixado ao padrão. A rota só será correspondida se o caminho URL for `/users/show`, e quando a função [`HttpRequest.url_for()`][urlfor] for chamada com o nome da rota `show_users`, ela gerará um URL com o mesmo caminho.

## Guardas de Aplicativo e hospedagem virtual

Você pode pensar em uma guarda como uma função simples que aceita uma referência a um objeto de _request_ e retorna _true_ ou _false_. Formalmente, uma guarda é qualquer objeto que implementa a trait [`Guard`][guardtrait]. O Actix Web fornece várias guardas. Você pode verificar a seção [funções][guardfuncs] da documentação da API.

Uma das guardas fornecidas é [`Host`][guardheader]. Ela pode ser usada como um filtro com base nas informações do cabeçalho da solicitação.

```rust
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(
                web::scope("/")
                    .guard(guard::Host("www.rust-lang.org"))
                    .route("", web::to(|| async { HttpResponse::Ok().body("www") })),
            )
            .service(
                web::scope("/")
                    .guard(guard::Host("users.rust-lang.org"))
                    .route("", web::to(|| async { HttpResponse::Ok().body("user") })),
            )
            .route("/", web::to(HttpResponse::Ok))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

## Configuração

Para simplicidade e reutilização, tanto [`App`][appconfig] quanto [`web::Scope`][webscopeconfig] fornecem o método `configure`. Essa função é útil para mover partes da configuração para um módulo diferente ou até mesmo uma biblioteca. Por exemplo, parte da configuração do recurso pode ser movida para um módulo diferente.

```rust
use actix_web::{web, App, HttpResponse, HttpServer};

// esta função poderia estar localizada em um módulo diferente
fn scoped_config(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::resource("/test")
            .route(web::get().to(|| async { HttpResponse::Ok().body("test") }))
            .route(web::head().to(HttpResponse::MethodNotAllowed)),
    );
}

// esta função poderia estar localizada em um módulo diferente
fn config(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::resource("/app")
            .route(web::get().to(|| async { HttpResponse::Ok().body("app") }))
            .route(web::head().to(HttpResponse::MethodNotAllowed)),
    );
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .configure(config)
            .service(web::scope("/api").configure(scoped_config))
            .route(
                "/",
                web::get().to(|| async { HttpResponse::Ok().body("/") }),
            )
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

O resultado do exemplo acima seria:

```
/         -> "/"
/app      -> "app"
/api/test -> "test"
```

Cada [`ServiceConfig`][serviceconfig] pode ter seus próprios `data`, `routes` e `services`.

<!-- LINKS -->

[usingappprefix]: ./../avancado/despacho-de-url.md#using-an-application-prefix-to-compose-applications
[stateexample]: https://github.com/actix/examples/blob/master/basics/state/src/main.rs
[guardtrait]: https://docs.rs/actix-web/4/actix_web/guard/trait.Guard.html
[guardfuncs]: https://docs.rs/actix-web/4/actix_web/guard/index.html#functions
[guardheader]: https://docs.rs/actix-web/4/actix_web/guard/fn.Header.html
[data]: https://docs.rs/actix-web/4/actix_web/web/struct.Data.html
[app]: https://docs.rs/actix-web/4/actix_web/struct.App.html
[appconfig]: https://docs.rs/actix-web/4/actix_web/struct.App.html#method.configure
[appdata]: https://docs.rs/actix-web/4/actix_web/struct.App.html#method.app_data
[scope]: https://docs.rs/actix-web/4/actix_web/struct.Scope.html
[webscopeconfig]: https://docs.rs/actix-web/4/actix_web/struct.Scope.html#method.configure
[webscope]: https://docs.rs/actix-web/4/actix_web/web/fn.scope.html
[urlfor]: https://docs.rs/actix-web/4/actix_web/struct.HttpRequest.html#method.url_for
[serviceconfig]: https://docs.rs/actix-web/4/actix_web/web/struct.ServiceConfig.html