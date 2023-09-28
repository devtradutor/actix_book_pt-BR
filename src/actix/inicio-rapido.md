# Início Rápido

Antes de começar a escrever um aplicativo actix, você precisará ter uma versão do Rust instalada.
Recomendamos que você use o rustup para instalar ou configurar tal versão.

## Instalar Rust

Antes de começarmos, precisamos instalar o Rust usando o instalador [rustup](https://rustup.rs/):

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Se você já tem o rustup instalado, execute este comando para garantir que você tenha a versão mais recente do Rust:

```bash
rustup update
```

O framework actix requer a versão 1.40.0 ou superior do Rust.

## Executando Exemplos

A maneira mais rápida de começar a experimentar o actix é clonar o repositório do actix e executar os exemplos incluídos no diretório examples/. O seguinte conjunto de comandos executa o exemplo `ping`:

```bash
git clone https://github.com/actix/actix
cd actix
cargo run --example ping
```

Verifique o diretório [examples/](https://github.com/actix/actix/tree/master/actix/examples) para mais exemplos.
