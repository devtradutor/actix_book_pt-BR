
# Opções Assíncronas

Temos vários projetos de exemplo que mostram o uso de adaptadores de banco de dados assíncronos:

- [Postgres](https://github.com/actix/examples/tree/master/databases/postgres)
- [SQLite](https://github.com/actix/examples/tree/master/databases/sqlite)
- [MongoDB](https://github.com/actix/examples/tree/master/databases/mongodb)

# Diesel

As versões atuais do Diesel (v1/v2) não suportam operações assíncronas, portanto, é importante usar a função [`web::block`][web-block] para transferir suas operações de banco de dados para a thread-pool de tempo de execução do Actix.

Você pode criar funções de ação que correspondam a todas as operações que seu aplicativo realizará no banco de dados.

```rust
#[derive(Debug, Insertable)]
#[diesel(table_name = self::schema::users)]
struct NewUser<'a> {
    id: &'a str,
    name: &'a str,
}

fn insert_new_user(
    conn: &mut SqliteConnection,
    user_name: String,
) -> diesel::QueryResult<User> {
    use crate::schema::users::dsl::*;

    // Criar modelo de inserção
    let uid = format!("{}", uuid::Uuid::new_v4());
    let new_user = NewUser {
        id: &uid,
        name: &user_name,
    };

    // Operações normais do Diesel
    diesel::insert_into(users)
        .values(&new_user)
        .execute(conn)
        .expect("Erro ao inserir pessoa");

    let user = users
        .filter(id.eq(&uid))
        .first::<User>(conn)
        .expect("Erro ao carregar a pessoa que foi inserida agora");

    Ok(user)
}
```

Agora você deve configurar o pool do banco de dados usando uma biblioteca como `r2d2`, que disponibiliza várias conexões de banco de dados para o seu aplicativo. Isso significa que vários manipuladores podem interagir com o banco de dados ao mesmo tempo e ainda aceitar novas conexões. Simplesmente, o pool será parte do estado do seu aplicativo. (Nesse caso, é benéfico não usar uma struct de invólucro de estado, pois o pool cuida do acesso compartilhado para você.)

```rust
type DbPool = r2d2::Pool<r2d2::ConnectionManager<SqliteConnection>>;

#[actix_web::main]
async fn main() -> io::Result<()> {
    // conecta ao banco de dados SQLite
    let manager = r2d2::ConnectionManager::<SqliteConnection>::new("app.db");
    let pool = r2d2::Pool::builder()
        .build(manager)
        .expect("database URL should be valid path to SQLite DB file");

    // Inicia o servidor HTTP na porta 8080
    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(pool.clone()))
            .route("/{name}", web::get().to(index))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

Agora, em um manipulador de solicitação, use o extrator `Data<T>` para obter o pool a partir do estado do aplicativo e obter uma conexão a partir dele. Isso fornece uma conexão de banco de dados de propriedade que pode ser passada para um fechamento [`web::block`][web-block]. Em seguida, basta chamar a função de ação com os argumentos necessários e usar o `.await` no resultado.

Este exemplo também mapeia o erro para um `HttpResponse` antes de usar o operador `?`, mas isso não é necessário se o seu tipo de erro de retorno implementar [`ResponseError`][response-error].

```rust
async fn index(
    pool: web::Data<DbPool>,
    name: web::Path<(String,)>,
) -> actix_web::Result<impl Responder> {
    let (name,) = name.into_inner();

    let user = web::block(move || {
        // Obter uma conexão do pool também é uma operação potencialmente bloqueante.
        // Portanto, deve ser chamada dentro do fechamento `web::block`.
        let mut conn = pool.get().expect("Não foi possível obter uma conexão do banco de dados a partir do pool.");

        insert_new_user(&mut conn, name)
    })
    .await?
    .map_err(error::ErrorInternalServerError)?;

    Ok(HttpResponse::Ok().json(user))
}
```

Isso é tudo! Veja o exemplo completo [aqui](https://github.com/actix/examples/tree/master/databases/diesel)

[web-block]: https://docs.rs/actix-web/4/actix_web/web/fn.block.html
[response-error]: https://docs.rs/actix-web/4/actix_web/trait.ResponseError.html