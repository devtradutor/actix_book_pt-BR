# Arquivo individual

É possível servir arquivos estáticos com um padrão de caminho personalizado e `NamedFile`. Para corresponder a uma parte do caminho, podemos usar uma expressão regular `[.*]`.

```rust
use actix_files::NamedFile;
use actix_web::{HttpRequest, Result};
use std::path::PathBuf;

async fn index(req: HttpRequest) -> Result<NamedFile> {
    let path: PathBuf = req.match_info().query("filename").parse().unwrap();
    Ok(NamedFile::open(path)?)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    use actix_web::{web, App, HttpServer};

    HttpServer::new(|| App::new().route("/{filename:.*}", web::get().to(index)))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```
> **Perigo**
>
> Corresponder a uma parte do caminho com a expressão regular `[.*]` e usá-la para retornar um `NamedFile` tem sérias implicações de segurança. 
> Isso oferece a possibilidade de um atacante inserir `../` na URL e acessar todos os arquivos no host aos quais o usuário que executa o servidor tem acesso.


## Diretório

Para servir arquivos de diretórios e subdiretórios específicos, pode-se usar [`Files`][files]. O `Files` deve ser registrado com o método `App::service()`, caso contrário, não será possível servir subcaminhos.

```rust
use actix_files as fs;
use actix_web::{App, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(fs::Files::new("/static", ".").show_files_listing()))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

Por padrão, a listagem de arquivos para subdiretórios está desativada. Uma tentativa de carregar a listagem de diretórios retornará uma resposta _404 Not Found_. Para habilitar a listagem de arquivos, use o método [`Files::show_files_listing()`][showfileslisting].

Em vez de mostrar a listagem de arquivos de um diretório, é possível redirecionar para um arquivo de índice específico. Use o método [`Files::index_file()`][indexfile] para configurar esse redirecionamento.

## Configuração

`NamedFiles` pode especificar várias opções para servir arquivos:

- `set_content_disposition` - função usada para mapear o tipo MIME do arquivo para o tipo `Content-Disposition` correspondente
- `use_etag` - especifica se o `ETag` deve ser calculado e incluído nos cabeçalhos.
- `use_last_modified` - especifica se o timestamp de modificação do arquivo deve ser usado e adicionado ao cabeçalho `Last-Modified`.

Todos os métodos acima são opcionais e fornecidos com as melhores configurações padrão, mas é possível personalizar qualquer um deles.

```rust
use actix_files as fs;
use actix_web::http::header::{ContentDisposition, DispositionType};
use actix_web::{get, App, Error, HttpRequest, HttpServer};

#[get("/{filename:.*}")]
async fn index(req: HttpRequest) -> Result<fs::NamedFile, Error> {
    let path: std::path::PathBuf = req.match_info().query("filename").parse().unwrap();
    let file = fs::NamedFile::open(path)?;
    Ok(file
        .use_last_modified(true)
        .set_content_disposition(ContentDisposition {
            disposition: DispositionType::Attachment,
            parameters: vec![],
        }))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(index))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

A configuração também pode ser aplicada ao serviço de diretório:

```rust
use actix_files as fs;
use actix_web::{App, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new().service(
            fs::Files::new("/static", ".")
                .show_files_listing()
                .use_last_modified(true),
        )
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

[files]: https://docs.rs/actix-files/0.6/actix_files/struct.Files.html#
[showfileslisting]: https://docs.rs/actix-files/0.6/actix_files/struct.Files.html#method.show_files_listing
[indexfile]: https://docs.rs/actix-files/0.6/actix_files/struct.Files.html#method.index_file