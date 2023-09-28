# Actix Web faz parte de um Ecossistema de Crates

Há muito tempo, o Actix Web foi construído em cima do framework de atores `actix`. Agora, o Actix Web é em grande parte independente do framework de atores e é construído usando um sistema diferente. Embora o `actix` ainda seja mantido, sua utilidade como uma ferramenta geral está diminuindo à medida que o ecossistema de futures e async/await amadurece. Neste momento, o uso do `actix` é necessário apenas para pontos de extremidade de WebSocket.

Chamamos o Actix Web de um framework poderoso e pragmático. Para todos os efeitos, é um micro-framework com algumas peculiaridades. Se você já é um programador Rust, provavelmente se sentirá em casa rapidamente, mas mesmo se estiver vindo de outra linguagem de programação, você deverá achar o Actix Web fácil de aprender.

Uma aplicação desenvolvida com o Actix Web expõe um servidor HTTP contido em um executável nativo. Você pode colocá-lo atrás de outro servidor HTTP como o nginx ou servi-lo como está. Mesmo na ausência completa de outro servidor HTTP, o Actix Web é poderoso o suficiente para fornecer suporte para HTTP/1 e HTTP/2, além de TLS (HTTPS). Isso o torna útil para construir pequenos serviços prontos para produção.


Mais importante ainda: o Actix Web roda na versão do Rust 1.59  ou posterior e funciona com versões estáveis.


[tokio]: https://tokio.rs