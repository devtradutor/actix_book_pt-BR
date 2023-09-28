## Instalando o Rust

Se você ainda não tem o Rust instalado, recomendamos que use o `rustup` para gerenciar a instalação. O [guia oficial do Rust][rustguide] possui uma seção maravilhosa sobre como começar.

O Actix Web atualmente requer a versão mínima suportada do Rust (MSRV) de  1.59. Executar o comando `rustup update` garantirá que você tenha a versão mais recente do Rust disponível. Portanto, este guia pressupõe que você esteja executando o Rust 1.59 ou posterior.

## Olá, mundo!

Comece criando um novo projeto Cargo baseado em binário e entre no novo diretório:

```bash
cargo new hello-world
cd hello-world
```

Adicione o `actix-web` como uma dependência do seu projeto, adicionando o seguinte ao arquivo `Cargo.toml`.

```toml
[dependencies]
actix-web = "4"
```

Os manipuladores de requisição usam funções assíncronas que aceitam zero ou mais parâmetros. Esses parâmetros podem ser extraídos de uma requisição (consulte a trait `FromRequest`) e retornam um tipo que pode ser convertido em uma `HttpResponse` (consulte a trait `Responder`):

Substitua o conteúdo de `src/main.rs` pelo seguinte:

```rust
use actix_web::{get, post, web, App, HttpResponse, HttpServer, Responder};

#[get("/")]
async fn hello() -> impl Responder {
    HttpResponse::Ok().body("Olá, mundo!")
}

#[post("/echo")]
async fn echo(req_body: String) -> impl Responder {
    HttpResponse::Ok().body(req_body)
}

async fn manual_hello() -> impl Responder {
    HttpResponse::Ok().body("Olá")
}
```

Observe que alguns desses manipuladores possuem informações de roteamento anexadas diretamente usando as macros embutidas. Isso permite especificar o método e o caminho que o manipulador deve responder. Você verá abaixo como registrar `manual_hello` (ou seja, rotas que não usam uma macro de roteamento).

Em seguida, crie uma instância de `App` e registre os manipuladores de requisição. Use `App::service` para os manipuladores que usam macros de roteamento e `App::route` para manipuladores roteados manualmente, declarando o caminho e o método. Por fim, o aplicativo é iniciado dentro de um `HttpServer`, que servirá as requisições recebidas usando seu `App` como uma "fábrica de aplicativos".

Acrescente a seguinte função `main` ao final de `src/main.rs`:

```rust 
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(hello)
            .service(echo)
            .route("/hey", web::get().to(manual_hello))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

É isso! Compile e execute o programa com `cargo run`. A macro `#[actix_web::main]` executa a função main assíncrona dentro do runtime do actix. Agora você pode acessar `http://127.0.0.1:8080/` ou qualquer outra rota que você definiu para ver os resultados.

<!-- LINKS -->

[rustguide]: https://doc.rust-lang.org/book/ch01-01-installation.html
[actix-web-codegen]: https://docs.rs/actix-web-codegen/