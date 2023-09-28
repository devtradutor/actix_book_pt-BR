# Despacho de URL

O despacho de URL fornece uma maneira simples de mapear URLs para código de manipulador usando uma linguagem de correspondência de padrões simples. Se um dos padrões corresponder às informações de caminho associadas a uma solicitação, um objeto manipulador específico é invocado.

> Um manipulador de solicitação é uma função que aceita zero ou mais parâmetros que podem ser extraídos de uma solicitação (ou seja, [_impl FromRequest_][implfromrequest]) e retorna um tipo que pode ser convertido em uma HttpResponse (ou seja, [_impl Responder_][implresponder]). Mais informações estão disponíveis na [seção de manipuladores][handlersection].

## Configuração de Recurso

A configuração de recursos é a ação de adicionar um novo recurso a uma aplicação. Um recurso possui um nome, que atua como um identificador a ser usado para a geração de URLs. O nome também permite que os desenvolvedores adicionem rotas a recursos existentes. Um recurso também possui um padrão, destinado a corresponder à parte `PATH` de uma URL (a parte que segue o esquema e a porta, por exemplo, `_/foo/bar_` na URL `_http://localhost:8080/foo/bar?q=value_`). Não corresponde à parte QUERY (a parte que segue o ?, por exemplo, `_q=value_` em `_http://localhost:8080/foo/bar?q=value_`).

O método [_App::route()_][approute] fornece uma maneira simples de registrar rotas. Este método adiciona uma única rota à tabela de roteamento da aplicação. Este método aceita _padrão de caminho_, um _método HTTP_ e uma _função manipuladora_. O método `route()` pode ser chamado várias vezes para o mesmo caminho; nesse caso, várias rotas são registradas para o mesmo caminho de recurso.

```rust
use actix_web::{web, App, HttpResponse, HttpServer};

async fn index() -> HttpResponse {
    HttpResponse::Ok().body("Olá")
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/", web::get().to(index))
            .route("/user", web::post().to(index))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

Embora o método _App::route()_ forneça uma maneira simples de registrar rotas, para acessar a configuração completa de recursos, um método diferente deve ser usado. O método [_App::service()_][appservice] adiciona um único [recurso][webresource] à tabela de roteamento da aplicação. Este método aceita um _padrão de caminho_, _guards_ e uma ou mais rotas.

```rust
use actix_web::{guard, web, App, HttpResponse};

async fn index() -> HttpResponse {
    HttpResponse::Ok().body("Olá")
}

pub fn main() {
    App::new()
        .service(web::resource("/prefix").to(index))
        .service(
            web::resource("/user/{name}")
                .name("user_detail")
                .guard(guard::Header("content-type", "application/json"))
                .route(web::get().to(HttpResponse::Ok))
                .route(web::put().to(HttpResponse::Ok)),
        );
}
```

Se um recurso não contiver nenhuma rota ou não tiver rotas correspondentes, ele retornará uma resposta HTTP de _NOT FOUND_.

### Configurando uma Rota

Um recurso contém um conjunto de rotas. Cada rota, por sua vez, possui um conjunto de `guards` (guardas) e um manipulador. Novas rotas podem ser criadas com o método `Resource::route()`, que retorna uma referência para uma nova instância de _Route_. Por padrão, a rota não contém guards(guardas) e, portanto, corresponde a todas as solicitações, e o manipulador padrão é `HttpNotFound`.

A aplicação roteia as solicitações recebidas com base nos critérios de rota que são definidos durante o registro do recurso e o registro da rota. O recurso corresponde a todas as rotas que contém na ordem em que as rotas foram registradas por meio do método `Resource::route()`.

> Uma _Route_ pode conter qualquer número de _guards_, mas apenas um manipulador.

```rust
App::new().service(
    web::resource("/path").route(
        web::route()
            .guard(guard::Get())
            .guard(guard::Header("content-type", "text/plain"))
            .to(HttpResponse::Ok),
    ),
)
```

Neste exemplo, `HttpResponse::Ok()` é retornado para solicitações _GET_ se a solicitação contiver o cabeçalho `Content-Type`, o valor deste cabeçalho for _text/plain_ e o caminho for igual a `/path`.

Se um recurso não puder corresponder a nenhuma rota, uma resposta "NOT FOUND" é retornada.

O método [_ResourceHandler::route()_][resourcehandler] retorna um objeto [_Route_][route]. A rota pode ser configurada com um padrão semelhante a um construtor. Os seguintes métodos de configuração estão disponíveis:

- [_Route::guard()_][routeguard] registra um novo guard. Qualquer número de guards pode ser registrado para cada rota.
- [_Route::method()_][routemethod] registra um método de guard. Qualquer número de guards pode ser registrado para cada rota.
- [_Route::to()_][routeto] registra uma função manipuladora assíncrona para esta rota. Apenas um manipulador pode ser registrado. Geralmente, o registro do manipulador é a última operação de configuração.

## Correspondência de Rotas

O principal objetivo da configuração de rotas é corresponder (ou não corresponder) ao `path` da solicitação em relação a um padrão de caminho de URL. O `path` representa a parte de caminho da URL que foi solicitada.

A forma como o _actix-web_ faz isso é muito simples. Quando uma solicitação entra no sistema, para cada declaração de configuração de recurso presente no sistema, o ato de verifica o caminho da solicitação em relação ao padrão declarado. Essa verificação ocorre na ordem em que as rotas foram declaradas por meio do método `App::service()`. Se um recurso não for encontrado, o _recurso padrão_ é usado como o recurso correspondente.

Quando uma configuração de rota é declarada, ela pode conter argumentos de guard de rota. Todos os guards de rota associados a uma declaração de rota devem ser `true` para que a configuração de rota seja usada para uma determinada solicitação durante uma verificação. Se qualquer guard no conjunto de argumentos de guardas de rota fornecidos a uma configuração de rota retornar `false` durante uma verificação, aquela rota é ignorada e a correspondência de rotas continua por meio do conjunto ordenado de rotas.

Se alguma rota corresponder, o processo de correspondência de rotas é interrompido e o manipulador associado à rota é invocado. Se nenhuma rota corresponder após todos os padrões de rota serem esgotados, uma resposta _NOT FOUND_ é retornada.

## Sintaxe do padrão de recursos

A sintaxe da linguagem de correspondência de padrões usada pelo Actix no argumento de padrão é direta.

O padrão usado na configuração de rota pode começar com um caractere de barra. Se o padrão não começar com um caractere de barra, uma barra implícita será adicionada a ele no momento da correspondência. Por exemplo, os seguintes padrões são equivalentes:

```
{foo}/bar/baz
```

e:

```
/{foo}/bar/baz
```

Uma parte _variável_ (marcador de substituição) é especificada no formato _{identificador}_, o que significa "aceitar quaisquer caracteres até o próximo caractere de barra e usá-lo como nome no objeto `HttpRequest.match_info()`".

Um marcador de substituição em um padrão corresponde à expressão regular `[^{}/]+`.

Um `match_info` é o objeto `Params` que representa as partes dinâmicas extraídas de uma URL com base no padrão de roteamento. Ele está disponível como `request.match_info`. Por exemplo, o padrão a seguir define um segmento literal (foo) e dois marcadores de substituição (baz e bar):

```
foo/{baz}/{bar}
```

O padrão acima corresponderá a estas URLs, gerando as seguintes informações de correspondência:

```
foo/1/2        -> Params {'baz': '1', 'bar': '2'}
foo/abc/def    -> Params {'baz': 'abc', 'bar': 'def'}
```

No entanto, não corresponderá aos seguintes padrões:

```
foo/1/2/        -> Sem correspondência (barra final)
bar/abc/def     -> Incompatibilidade literal no primeiro segmento
```

A correspondência para um marcador de substituição de segmento será feita apenas até o primeiro caractere não alfanumérico no segmento do padrão. Por exemplo, se este padrão de rota for usado:

```
foo/{name}.html
```

O caminho literal `/foo/biz.html` corresponderá ao padrão de rota acima, e o resultado da correspondência será `Params {'name': 'biz'}`. No entanto, o caminho literal `/foo/biz` não corresponderá, porque ele não contém uma parte literal `.html` no final do segmento representado por `{name}.html` (ele contém apenas biz, não biz.html).

Para capturar ambos os segmentos, dois marcadores de substituição podem ser usados:

```
foo/{name}.{ext}
```

O caminho literal `/foo/biz.html` corresponderá ao padrão de rota acima, e o resultado da correspondência será `Params {'name': 'biz', 'ext': 'html'}`. Isso ocorre porque há uma parte literal de `.` (ponto) entre os dois marcadores de substituição `{name}` e `{ext}`.

Os marcadores de substituição podem opcionalmente especificar uma expressão regular que será usada para decidir se um segmento de caminho deve corresponder ao marcador. Para especificar que um marcador de substituição deve corresponder apenas a um conjunto específico de caracteres definidos por uma expressão regular, você deve usar uma forma ligeiramente estendida da sintaxe de marcador de substituição. Dentro das chaves, o nome do marcador de substituição deve ser seguido por dois pontos e, em seguida, imediatamente, a expressão regular. A expressão regular padrão associada a um marcador de substituição `[^/]+` corresponde a um ou mais caracteres que não sejam uma barra. Por exemplo, internamente, o marcador de substituição `{foo}` pode ser expresso de forma mais detalhada como `{foo:[^/]+}`. Você pode alterar isso para ser uma expressão regular arbitrária para corresponder a uma sequência arbitrária de caracteres, como `{foo:\d+}` para corresponder apenas a dígitos.

Os segmentos devem conter pelo menos um caractere para corresponder a um marcador de substituição de segmento. Por exemplo, para a URL `/abc/`:

- `/abc/{foo}` não corresponderá.
- `/{foo}/` corresponderá.

> **Observação**: o caminho será descodificado e convertido em uma string Unicode válida antes de corresponder ao padrão, e os valores que representam os segmentos de caminho correspondidos também serão decodificados.

Portanto, por exemplo, o padrão a seguir:

```
foo/{bar}
```

Ao corresponder à seguinte URL:

```
http://example.com/foo/La%20Pe%C3%B1a
```

O dicionário de correspondência será assim (o valor está decodificado da URL):

```
Params {'bar': 'La Pe\xf1a'}
```

As strings literais no segmento do caminho devem representar o valor decodificado do caminho fornecido ao Actix. Não se deve usar um valor codificado em URL no padrão. Por exemplo, em vez disso:

```
/Foo%20Bar/{baz}
```

Deve-se usar algo como:

```
/Foo Bar/{baz}
```

É possível obter uma "correspondência de cauda". Para isso, uma expressão regular personalizada deve ser usada.

```
foo/{bar}/{tail:.*}
```

O padrão acima corresponderá a estas URLs, gerando as seguintes informações de correspondência:

```
foo/1/2/           -> Params {'bar': '1', 'tail': '2/'}
foo/abc/def/a/b/c  -> Params {'bar': 'abc', 'tail': 'def/a/b/c'}
```

##  Escopo de Rotas

O escopo ajuda você a organizar rotas compartilhando caminhos raiz comuns. Você pode aninhar escopos dentro de escopos.

Suponha que você queira organizar os caminhos para endpoints usados para visualizar "Usuários". Esses caminhos podem incluir:

- /users
- /users/show
- /users/show/{id}

Um layout em escopo desses caminhos seria o seguinte

```rust
#[get("/show")]
async fn show_users() -> HttpResponse {
    HttpResponse::Ok().body("Mostrar usuários")
}

#[get("/show/{id}")]
async fn user_detail(path: web::Path<(u32,)>) -> HttpResponse {
    HttpResponse::Ok().body(format!("Detalhes do usuário: {}", path.into_inner().0))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new().service(
            web::scope("/users")
                .service(show_users)
                .service(user_detail),
        )
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

Um caminho _em escopo_ pode conter segmentos de caminho variáveis como recursos. Isso é consistente com caminhos não escopados.

Você pode obter segmentos de caminho variáveis a partir de `HttpRequest::match_info()`. O [extrator de `caminho`][pathextractor] também é capaz de extrair segmentos de escopo de nível variável.

## Informações de correspondência

Todos os valores que representam os segmentos de caminho correspondidos estão disponíveis em [`HttpRequest::match_info`][matchinfo]. Valores específicos podem ser obtidos com [`Path::get()`][pathget].

```rust
use actix_web::{get, App, HttpRequest, HttpServer, Result};

#[get("/a/{v1}/{v2}/")]
async fn index(req: HttpRequest) -> Result<String> {
    let v1: u8 = req.match_info().get("v1").unwrap().parse().unwrap();
    let v2: u8 = req.match_info().query("v2").parse().unwrap();
    let (v3, v4): (u8, u8) = req.match_info().load().unwrap();
    Ok(format!("Values {} {} {} {}", v1, v2, v3, v4))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(index))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

Para este exemplo, para o caminho '/a/1/2/', os valores v1 e v2 serão resolvidos como "1" e "2".

### Extrator de informações de caminho

O Actix fornece funcionalidade para extração de informações de caminho de forma segura por tipo. O [_Path_][pathstruct] extrai informações, e o tipo de destino pode ser definido de várias formas diferentes. A abordagem mais simples é usar um tipo `tuple`. Cada elemento na tupla deve corresponder a um elemento do padrão de caminho. Por exemplo, você pode corresponder o padrão de caminho `/{id}/{username}/` ao tipo `Path<(u32, String)>`, mas o tipo `Path<(String, String, String)>` sempre falhará.

```rust
use actix_web::web;

async fn index(info: web::Path<(u32, String)>) -> String {
    format!("Hello {}! ID: {}", info.1, info.0)
}
```

Também é possível extrair informações do padrão de caminho para uma estrutura. Nesse caso, essa estrutura deve implementar o traço `Deserialize` do _serde_.

```rust
use actix_web::{web, Result};
use serde::Deserialize;

#[derive(Deserialize)]
struct Info {
    id: u32,
    username: String,
}

async fn index(info: web::Path<Info>) -> Result<String> {
    Ok(format!("Hello {}! ID: {}", info.username, info.id))
}
```

O [_Query_][query] fornece funcionalidade semelhante para os parâmetros de consulta da solicitação.

## Gerando URLs de recursos

Use o método [_HttpRequest.url_for()_][urlfor] para gerar URLs com base em padrões de recursos. Por exemplo, se você configurou um recurso com o nome "foo" e o padrão "{a}/{b}/{c}", você pode fazer o seguinte:

```rust
use actix_web::{get, guard, http::header, HttpRequest, HttpResponse, Result};

#[get("/test/")]
async fn index(req: HttpRequest) -> Result<HttpResponse> {
    let url = req.url_for("foo", ["1", "2", "3"])?; // <- Gerar URL para o recurso "foo"

    Ok(HttpResponse::Found()
        .insert_header((header::LOCATION, url.as_str()))
        .finish())
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    use actix_web::{web, App, HttpServer};

    HttpServer::new(|| {
        App::new()
            .service(
                web::resource("/test/{a}/{b}/{c}")
                    .name("foo") // <- Defina o nome do recurso e então ele poderá ser usado em `url_for`.
                    .guard(guard::Get())
                    .to(HttpResponse::Ok),
            )
            .service(index)
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

Isso retornaria algo como a string `http://example.com/test/1/2/3` (pelo menos se o protocolo e o nome do host atuais implicarem em http://example.com). O método `url_for()` retorna um [_objeto Url_][urlobj], para que você possa modificar essa URL (adicionar parâmetros de consulta, âncora, etc). `url_for()` só pode ser chamado para recursos com nomes, caso contrário, ocorrerá um erro.

## Recursos externos

Recursos que são URLs válidas podem ser registrados como recursos externos. Eles são úteis apenas para geração de URLs e nunca são considerados para correspondência no momento da solicitação.

```rust
use actix_web::{get, App, HttpRequest, HttpServer, Responder};

#[get("/")]
async fn index(req: HttpRequest) -> impl Responder {
    let url = req.url_for("youtube", ["oHg5SJYRHA0"]).unwrap();
    assert_eq!(url.as_str(), "https://youtube.com/watch/oHg5SJYRHA0");

    url.to_string()
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(index)
            .external_resource("youtube", "https://youtube.com/watch/{video_id}")
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

## Normalização de caminho e redirecionamento para rotas com barra no final

Por normalização, significa:

- Adicionar uma barra no final do caminho.
- Substituir várias barras por uma única barra.

O manipulador retorna assim que encontra um caminho que é resolvido corretamente. A ordem das condições de normalização, se todas estiverem habilitadas, é 1) mesclar, 2) mesclar e anexar e 3) anexar. Se o caminho for resolvido com pelo menos uma dessas condições, ele será redirecionado para o novo caminho.

```rust
use actix_web::{middleware, HttpResponse};

async fn index() -> HttpResponse {
    HttpResponse::Ok().body("Olá")
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    use actix_web::{web, App, HttpServer};

    HttpServer::new(|| {
        App::new()
            .wrap(middleware::NormalizePath::default())
            .route("/resource/", web::to(index))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

Neste exemplo, `//resource///` será redirecionado para `/resource/`.

Neste exemplo, o manipulador de normalização de caminho está registrado para todos os métodos, mas você não deve depender desse mecanismo para redirecionar solicitações _POST_. O redirecionamento da _Not Found_ com a adição da barra transformará uma solicitação _POST_ em GET, perdendo todos os dados _POST_ da solicitação original.

É possível registrar a normalização de caminho apenas para solicitações _GET_:

```rust
use actix_web::{get, http::Method, middleware, web, App, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .wrap(middleware::NormalizePath::default())
            .service(index)
            .default_service(web::route().method(Method::GET))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

## Usando um prefixo de aplicativo para compor aplicações

O método `web::scope()` permite definir um escopo específico para a aplicação. Esse escopo representa um prefixo de recurso que será adicionado a todos os padrões de recurso adicionados pela configuração do recurso. Isso pode ser usado para montar um conjunto de rotas em um local diferente do pretendido pelo autor da chamada, enquanto mantém os mesmos nomes de recurso.

Por exemplo:

```rust
#[get("/show")]
async fn show_users() -> HttpResponse {
    HttpResponse::Ok().body("Mostrar usuários")
}

#[get("/show/{id}")]
async fn user_detail(path: web::Path<(u32,)>) -> HttpResponse {
    HttpResponse::Ok().body(format!("Detalhes do usuário: {}", path.into_inner().0))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new().service(
            web::scope("/users")
                .service(show_users)
                .service(user_detail),
        )
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

No exemplo acima, a rota _show_users_ terá um padrão de rota efetivo de _/users/show_ em vez de _/show_, porque o escopo da aplicação será adicionado ao padrão. A rota só será correspondida se o caminho URL for _/users/show_ e, quando a função `HttpRequest.url_for()` for chamada com o nome da rota show_users, ela gerará uma URL com esse mesmo caminho.

## Guard de rota personalizada

Você pode pensar em uma guarda como uma função simples que aceita uma referência de objeto _request_ e retorna _true_ ou _false_. Formalmente, uma guardd é qualquer objeto que implementa a trait [`Guard`][guardtrait]. O Actix fornece vários predicados, você pode verificar a [seção de funções][guardfuncs] da documentação da API.

Aqui está um exemplo simples de guard que verifica se uma solicitação contém um _header_ específico:

```rust
use actix_web::{
    guard::{Guard, GuardContext},
    http, HttpResponse,
};

struct ContentTypeHeader;

impl Guard for ContentTypeHeader {
    fn check(&self, req: &GuardContext) -> bool {
        req.head()
            .headers()
            .contains_key(http::header::CONTENT_TYPE)
    }
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    use actix_web::{web, App, HttpServer};

    HttpServer::new(|| {
        App::new().route(
            "/",
            web::route().guard(ContentTypeHeader).to(HttpResponse::Ok),
        )
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

Neste exemplo, o manipulador _index_ só será chamado se a solicitação contiver o _header_ _CONTENT-TYPE_.

As guards não podem acessar ou modificar o objeto de solicitação, mas é possível armazenar informações extras em [extensões de solicitação][requestextensions].

## Modificando valores da guard

Você pode inverter o significado de qualquer valor de predicado envolvendo-o em um predicado `Not`. Por exemplo, se você deseja retornar uma resposta "METHOD NOT ALLOWED" para todos os métodos, exceto "GET":

```rust
use actix_web::{guard, web, App, HttpResponse, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new().route(
            "/",
            web::route()
                .guard(guard::Not(guard::Get()))
                .to(HttpResponse::MethodNotAllowed),
        )
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

A guard `Any` aceita uma lista de guards e corresponde se qualquer uma das guards fornecidas corresponder. Por exemplo:

```rust
guard::Any(guard::Get()).or(guard::Post())
```

A guard `All` aceita uma lista de guards e corresponde se todas as guards fornecidas corresponderem. Por exemplo:

```rust
guard::All(guard::Get()).and(guard::Header("content-type", "plain/text"))
```

## Alterando a resposta padrão de "Não encontrado"

Se o padrão de caminho não puder ser encontrado na tabela de roteamento ou se um recurso não encontrar uma rota correspondente, o recurso padrão é utilizado. A resposta padrão é "NÃO ENCONTRADO" (_NOT FOUND_). É possível substituir a resposta " NOT FOUND" com o método `App::default_service()`. Esse método aceita uma _função de configuração_ igual à configuração normal de recursos com o método `App::service()`.

```rust
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(web::resource("/").route(web::get().to(index)))
            .default_service(
                web::route()
                    .guard(guard::Not(guard::Get()))
                    .to(HttpResponse::MethodNotAllowed),
            )
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

[handlersection]: ./../fundamentos/manipuladores.md
[approute]: https://docs.rs/actix-web/4/actix_web/struct.App.html#method.route
[appservice]: https://docs.rs/actix-web/4/actix_web/struct.App.html?search=#method.service
[webresource]: https://docs.rs/actix-web/4/actix_web/struct.Resource.html
[resourcehandler]: https://docs.rs/actix-web/4/actix_web/struct.Resource.html#method.route
[route]: https://docs.rs/actix-web/4/actix_web/struct.Route.html
[routeguard]: https://docs.rs/actix-web/4/actix_web/struct.Route.html#method.guard
[routemethod]: https://docs.rs/actix-web/4/actix_web/struct.Route.html#method.method
[routeto]: https://docs.rs/actix-web/4/actix_web/struct.Route.html#method.to
[matchinfo]: https://docs.rs/actix-web/4/actix_web/struct.HttpRequest.html#method.match_info
[pathget]: https://docs.rs/actix-web/4/actix_web/dev/struct.Path.html#method.get
[pathstruct]: https://docs.rs/actix-web/4/actix_web/dev/struct.Path.html
[query]: https://docs.rs/actix-web/4/actix_web/web/struct.Query.html
[urlfor]: https://docs.rs/actix-web/4/actix_web/struct.HttpRequest.html#method.url_for
[urlobj]: https://docs.rs/url/1.7.2/url/struct.Url.html
[guardtrait]: https://docs.rs/actix-web/4/actix_web/guard/trait.Guard.html
[guardfuncs]: https://docs.rs/actix-web/4/actix_web/guard/index.html#functions
[requestextensions]: https://docs.rs/actix-web/4/actix_web/struct.HttpRequest.html#method.extensions
[implfromrequest]: https://docs.rs/actix-web/4/actix_web/trait.FromRequest.html
[implresponder]: https://docs.rs/actix-web/4/actix_web/trait.Responder.html
[pathextractor]: ./../fundamentos/extratores.md