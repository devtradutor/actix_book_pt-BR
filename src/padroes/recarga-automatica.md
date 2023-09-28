# Servidor de Desenvolvimento com Recarga Automática(Auto-Reloading)

Durante o desenvolvimento, pode ser muito útil ter o cargo recompilando automaticamente o código quando houver alterações. Isso pode ser facilmente realizado usando o [`cargo-watch`].

```sh
 cargo watch -x run
 ```

## Nota Histórica

Uma versão antiga desta página recomendava o uso de uma combinação de systemfd e listenfd, mas isso apresentava várias armadilhas e era difícil de integrar corretamente, especialmente quando fazia parte de um fluxo de trabalho de desenvolvimento mais amplo. Consideramos [`cargo-watch`] suficiente para fins de recarga automática.

[`cargo-watch`]: https://github.com/passcod/cargo-watch