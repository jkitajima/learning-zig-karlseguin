# Aprendendo Zig

Bem-vindo(a) ao livro Aprendendo Zig, uma introdução à linguagem de programação Zig. Este guia tem como objetivo familiarizá-lo(a) com Zig. Ele presume experiência prévia em programação, embora não em uma linguagem específica.

Zig está em desenvolvimento intenso, e tanto a linguagem quanto a sua biblioteca padrão estão em constante evolução. Este guia se destina à versão de desenvolvimento mais recente do Zig. No entanto, é possível que parte do código esteja desatualizada. Se você baixou a versão mais recente da linguagem Zig e encontrou problemas ao executar algum código, por favor, [relate o problema](https://github.com/karlseguin/blog/issues)[^1].



## Traduções

- [Chinês](https://zigcc.github.io/learning-zig/index.html) - por Jiacai Liu
- [Russo](https://github.com/dee0xeed/learning-zig-rus/blob/main/src/ch01.md) - por dee0xeed
- [Coreano](https://faultnote.github.io/posts/learning-zig/) - por faultnote




## Índice

1. [Instalação](#instalação)
2. [Visão Geral da Linguagem - Parte 1](./01-language_overview_part_1.md)
3. [Visão Geral da Linguagem - Parte 2](./02-language_overview_part_2.md)
4. [Convenções de Estilização](./03-style_guide.md)
5. [Ponteiros](./04-pointers.md)
6. [Memória de Pilha](./05-stack_memory.md)
7. [Memória Dinâmica & Alocadores](./06-heap_memory_and_allocators.md)
8. [Genéricos (parametrização polimórfica)](./07-generics.md)
9. [Codificando em Zig](./08-coding_in_zig.md)
10. [Conclusão](./09-conclusion.md)



## Instalação

A página de [_download_](https://ziglang.org/download/) do Zig inclui binários pré-compilados para plataformas comuns. Nesta página, você encontrará binários para a versão de desenvolvimento mais recente, bem como para as principais versões. A versão mais recente, seguida por este guia, pode ser encontrada no topo da página.

Para o meu computador, vou baixar **_zig-macos-aarch64-0.12.0-dev.161+6a5463951.tar.xz_**. Você pode estar em uma plataforma diferente ou em uma versão mais recente. Após extrair o arquivo, você deve ter um binário `zig` (além de outras coisas) que desejará criar um alias ou adicionar ao seu caminho; de acordo com o fluxo ao qual está acostumado.

Agora, você deve conseguir executar no terminal os comandos `zig zen` e `zig version` para testar sua configuração.



[^1]: Relatar o problema em inglês.
