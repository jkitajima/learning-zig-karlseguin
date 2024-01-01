# Ponteiros

Zig não inclui um coletor de lixo. A responsabilidade de gerenciar a memória recai sobre você, o programador. É uma grande responsabilidade, pois tem impacto direto no desempenho, estabilidade e segurança da sua aplicação.

Começaremos falando sobre ponteiros, que é um tópico importante para discutir por si só, mas também para começar a nos treinar para ver os dados do nosso programa de um ponto de vista orientado à memória. Se você já está familiarizado com ponteiros, alocações de memória _heap_ e ponteiros pendentes _(dangling pointers)_, sinta-se à vontade para pular para os próximos capítulos.

---

O código a seguir cria um usuário com um `power` de 1 e, em seguida, chama a função `levelUp` que incrementa o poder do usuário em 1. Você consegue adivinhar a saída?

```zig
const std = @import("std");

pub fn main() void {
	var user = User{
		.id = 1,
		.power = 100,
	};

	// esta foi a linha adicionada
	levelUp(user);
	std.debug.print("User {d} has power of {d}\n", .{user.id, user.power});
}

fn levelUp(user: User) void {
	user.power += 1;
}

pub const User = struct {
	id: u64,
	power: i32,
};
```

Isso foi uma pegadinha; o código não será compilado: "cannot assign to constant" (não é possível atribuir a uma constante). Vimos na parte 1 que os parâmetros de função são constantes, portanto, `user.power += 1;` não é válido. Para corrigir o erro de compilação, poderíamos modificar a função `levelUp` para:

```zig
fn levelUp(user: User) void {
	var u = user;
	u.power += 1;
}
```

O código será compilado, mas a saída será "User 1 has power of 100", embora a intenção do nosso código seja aumentar o poder do usuário (função `levelUp`) para `101`. O que está acontecendo?

Para entender, ajuda pensar sobre dados em relação à memória e variáveis como rótulos que associam um tipo a um local específico na memória. Por exemplo, em `main`, criamos um `User`. Uma visualização simples desses dados na memória seria:

```
user -> ------------ (id)
        |    1     |
        ------------ (power)
        |   100    |
        ------------
```

Existem duas coisas importantes a serem observadas. A primeira é que nossa variável `user` aponta para o início de nossa estrutura. A segunda é que os campos são dispostos sequencialmente. Lembre-se de que nosso `user` também tem um tipo. Esse tipo nos diz que `id` é um inteiro de 64 bits e `power` é um inteiro de 32 bits. Armado com uma referência ao início de nossos dados e o tipo, o compilador pode traduzir `user.power` para: acessar um inteiro de 32 bits localizado 64 bits a partir do início. Isso é o poder das variáveis, elas referenciam a memória e incluem as informações de tipo necessárias para entender e manipular a memória de maneira significativa.

> Por padrão, Zig não faz garantias sobre o layout de memória das estruturas. Pode armazenar os campos em ordem alfabética, por tamanho ascendente ou com lacunas. Pode fazer o que quiser, desde que seja capaz de traduzir nosso código corretamente. Essa liberdade pode permitir certas otimizações. Somente se declararmos uma struct compacta (`packed struct`) obteremos garantias firmes sobre o layout de memória. Ainda assim, nossa visualização de `user` é razoável e útil.

Aqui está uma visualização um pouco diferente que inclui endereços de memória. O endereço de memória do início desses dados é um endereço aleatório que inventei. Este é o endereço de memória referenciado pela variável `user`, que também é o valor do nosso primeiro campo, `id`. No entanto, dado este endereço inicial, todos os endereços subsequentes têm um endereço relativo conhecido. Como `id` é um inteiro de 64 bits, ele ocupa 8 bytes de memória. Portanto, `power` deve estar em _$start_address + 8_:

```
user ->   ------------  (id: 1043368d0)
          |    1     |
          ------------  (power: 1043368d8)
          |   100    |
          ------------
```

Para verificar isso por si mesmo, gostaria de apresentar o operador _addressof_ (endereço de memória da variável): `&`. Como o nome indica, o operador `&` retorna o endereço de uma variável (também pode retornar o endereço de uma função, não é incrível?!). Mantendo a definição existente de `User`, experimente este código no `main`:

```zig
pub fn main() void {
	var user = User{
		.id = 1,
		.power = 100,
	};
	std.debug.print("{*}\n{*}\n{*}\n", .{&user, &user.id, &user.power});
}
```

Este código imprime o endereço de `user`, `user.id` e `user.power`. Você pode obter resultados diferentes com base em sua plataforma e outros fatores, mas espero que você veja que o endereço de `user` e `user.id` é o mesmo, enquanto `user.power` está em um deslocamento de 8 bytes. Eu obtive:

```
learning.User@1043368d0
u64@1043368d0
i32@1043368d8
```

O operador `&` retorna um ponteiro para um valor. Um ponteiro para um valor é um tipo distinto. O endereço de um valor do tipo `T` é um `*T`. Pronunciamos isso como um ponteiro para T. Portanto, se pegarmos o endereço de `user`, obteremos um `*User`, ou seja, um ponteiro para `User`:

```zig
pub fn main() void {
	var user = User{
		.id = 1,
		.power = 100,
	};

	const user_p = &user;
	std.debug.print("{any}\n", .{@TypeOf(user_p)});
}
```

Nosso objetivo original era aumentar o atributo `power` do nosso `user` em 1, por meio da função `levelUp`. Conseguimos fazer o código compilar, mas quando imprimimos o valor de `power`, ele ainda era o valor original. É um pouco avançado, mas vamos mudar o código para imprimir o endereço de `user` em `main` e em `levelUp`:

```zig
pub fn main() void {
	const user = User{
		.id = 1,
		.power = 100,
	};

	// linha acrescentada
	std.debug.print("main: {*}\n", .{&user});

	levelUp(user);
	std.debug.print("User {d} has power of {d}\n", .{user.id, user.power});
}

fn levelUp(user: User) void {
	// acrescente esta linha
	std.debug.print("levelUp: {*}\n", .{&user});
	var u = user;
	u.power += 1;
}
```

Se você executar isso, obterá dois endereços diferentes. Isso significa que `user` que está sendo modificado em `levelUp` é diferente de `user` em `main`. Isso acontece porque o Zig passa uma cópia do valor _(pass-by-value)_. Isso pode parecer uma escolha padrão estranha, mas um dos benefícios é que o chamador de uma função pode ter certeza de que a função não modificará o parâmetro (porque não pode). Em muitos casos, isso é uma garantia útil. Claro, às vezes, como com `levelUp`, queremos que a função modifique um parâmetro. Para conseguir isso, precisamos que `levelUp` atue no `user` real em `main`, não em uma cópia. Podemos fazer isso passando o endereço do nosso usuário para a função:

```zig
const std = @import("std");

pub fn main() void {
	var user = User{
		.id = 1,
		.power = 100,
	};

	// user -> &user
	levelUp(&user);
	std.debug.print("User {d} has power of {d}\n", .{user.id, user.power});
}

// User -> *User
fn levelUp(user: *User) void {
	user.power += 1;
}

pub const User = struct {
	id: u64,
	power: i32,
};
```

Tivemos que fazer duas alterações. A primeira foi chamar `levelUp` com o endereço de `user`, ou seja, `&user`, em vez de `user`. Isso significa que nossa função não recebe mais um `User`. Em vez disso, ela recebe um `*User`, que foi a nossa segunda alteração.

O código agora funciona conforme pretendido. Ainda há muitas sutilezas com os parâmetros de função e nosso modelo de memória em geral, mas estamos progredindo. Agora pode ser um bom momento para mencionar que, além da sintaxe específica, nada disso é exclusivo do Zig. O modelo que estamos explorando aqui é o mais comum; algumas linguagens podem apenas esconder muitos dos detalhes, e, assim, a flexibilidade, dos desenvolvedores.



## Métodos

Muito provavelmente, você teria escrito `levelUp` como um método da estrutura `User`:

```zig
pub const User = struct {
	id: u64,
	power: i32,

	fn levelUp(user: *User) void {
		user.power += 1;
	}
};
```

Isso levanta a pergunta: como chamamos um método com um receptor de ponteiro? Talvez tenhamos que fazer algo como: `&user.levelUp()`? Na verdade, você apenas o chama normalmente, ou seja, `user.levelUp()`. O Zig sabe que o método espera um ponteiro e passa o valor corretamente (por referência).

Inicialmente, escolhi uma função porque é explícita e, portanto, mais fácil de entender.



## Parâmetros constantes de função

Eu mais do que insinuei que, por padrão, o Zig passará uma cópia de um valor (chamado de "passagem por valor"). Em breve, veremos que a realidade é um pouco mais sutil (dica: e quanto a valores complexos com objetos aninhados?)

Mesmo ficando com tipos simples, a verdade é que o Zig pode passar parâmetros da maneira que quiser, contanto que possa garantir que a intenção do código seja preservada. Em nosso `levelUp` original, onde o parâmetro era um `User`, o Zig poderia ter passado uma cópia do usuário ou uma referência a `main.user`, contanto que pudesse garantir que a função não o modificasse. (Eu sei que, no final, queríamos que fosse modificado, mas ao fazer o tipo `User`, estávamos dizendo ao compilador que não queríamos).

Essa liberdade permite que o Zig use a estratégia mais otimizada com base no tipo do parâmetro. Tipos pequenos, como `User`, podem ser facilmente passados por valor (ou seja, copiados). Tipos maiores podem ser mais baratos de passar por referência. O Zig pode usar qualquer abordagem, contanto que a intenção do código seja preservada. Em certa medida, isso é possibilitado pela presença de parâmetros de função constantes.

Agora você sabe uma das razões pelas quais os parâmetros de função são constantes.

> Talvez você esteja se perguntando como passar por referência poderia ser mais lento, mesmo em comparação com a cópia de uma estrutura realmente pequena. Veremos isso de forma mais clara a seguir, mas a ideia principal é que acessar `user.power` quando `user` é um ponteiro adiciona um pequeno overhead. O compilador precisa avaliar o custo da cópia em comparação com o custo de acessar campos indiretamente por meio de um ponteiro.



## Ponteiros de ponteiros

Nós vimos anteriormente como era a memória de `user` dentro da nossa função `main`. Agora que alteramos o `levelUp`, como seria a memória dele?

```
main:
user -> ------------  (id: 1043368d0)  <---
        |    1     |                      |
        ------------  (power: 1043368d8)  |
        |   100    |                      |
        ------------                      |
                                          |
        .............  espaço vazio       |
        .............  ou outros dados    |
                                          |
levelUp:                                  |
user -> -------------  (*User)            |
        | 1043368d0 |----------------------
        -------------
```

Dentro de `levelUp`, `user` é um ponteiro para um `User`. Seu valor é um endereço. Não é apenas qualquer endereço, é claro, mas o endereço de `main.user`. Vale a pena ser explícito que a variável `user` em `levelUp` representa um valor concreto. Esse valor acontece de ser um endereço. E não é apenas um endereço, é também um tipo, um `*User`. É tudo muito consistente, não importa se estamos falando de ponteiros ou não: variáveis associam informações de tipo a um endereço. A única coisa especial sobre ponteiros é que, quando usamos a sintaxe de ponto, por exemplo, `user.power`, o Zig, sabendo que `user` é um ponteiro, seguirá automaticamente o endereço.

> Algumas outras linguagens requerem um símbolo diferente ao acessar um campo por meio de um ponteiro.

O importante de entender é que a variável `user` em `levelUp` em si existe na memória em algum endereço. Assim como fizemos antes, podemos verificar isso por nós mesmos:

```zig
fn levelUp(user: *User) void {
	std.debug.print("{*}\n{*}\n", .{&user, user});
	user.power += 1;
}
```

O código acima imprime o endereço que a variável `user` referencia, bem como o seu valor, que é o endereço de `user` em `main`.

Se `user` é um `*User`, então o que é `&user`? É um `**User`, ou seja, **um ponteiro para um ponteiro** para um `User`. Eu posso continuar fazendo isso até que um de nós fique sem memória!

**Existem casos de uso** para múltiplos níveis de indireção _(pointer indirection)_, mas não é algo que precisamos agora. O propósito desta seção é mostrar que ponteiros não são algo especial; eles são apenas um valor, que é um endereço, e um tipo.



## Ponteiros aninhados

Até agora, nosso `User` foi simples, contendo dois inteiros. É fácil visualizar sua memória e, quando falamos sobre "copiar", não há ambiguidade. Mas o que acontece quando `User` se torna mais complexo e contém um ponteiro?

```zig
pub const User = struct {
	id: u64,
	power: i32,
	name: []const u8,
};
```

Adicionamos um campo chamado `name`, que é um slice. Lembre-se de que um slice é composto por um comprimento e um ponteiro. Se inicializarmos nosso `user` com o nome `"Goku"`, como isso seria representado na memória?

```
user -> -------------  (id: 1043368d0)
        |     1     |
        -------------  (power: 1043368d8)
        |    100    |
        -------------  (name.len: 1043368dc)
        |     4     |
        -------------  (name.ptr: 1043368e4)
  ------| 1182145c0 |
  |     -------------
  |
  |     .............  espaço vazio
  |     .............  ou outros dados
  |
  --->  -------------  (1182145c0)
        |    'G'    |
        -------------
        |    'o'    |
        -------------
        |    'k'    |
        -------------
        |    'u'    |
        -------------
```

O novo campo `name` é um slice composto por campos `len` e `ptr`. Eles são dispostos em sequência junto com todos os outros campos. Em uma plataforma de 64 bits, tanto `len` quanto `ptr` serão de 64 bits, ou seja, 8 bytes. A parte interessante é o valor de `name.ptr`: é um endereço para algum outro lugar na memória.

> Já que usamos uma string literal, `user.name.ptr` apontará para uma localização específica dentro da área onde todas as constantes são armazenadas dentro do nosso binário.

Os tipos podem se tornar muito mais complexos do que isso com aninhamento profundo. Mas simples ou complexos, todos se comportam da mesma forma. Especificamente, se voltarmos ao nosso código original onde `levelUp` aceitava um simples `User` e Zig fornecia uma cópia, como isso seria agora que temos um ponteiro aninhado?

A resposta é que apenas uma cópia rasa do valor é feita. Ou, como alguns dizem, apenas a memória imediatamente acessível pela variável é copiada. Pode parecer que `levelUp` obteria uma cópia incompleta de `user`, possivelmente com um `name` inválido. Mas lembre-se de que um ponteiro, como nosso `user.name.ptr`, é um valor, e esse valor é um endereço. Uma cópia de um endereço ainda é o mesmo endereço:

```
main: user ->    -------------  (id: 1043368d0)
                 |     1     |
                 -------------  (power: 1043368d8)
                 |    100    |
                 -------------  (name.len: 1043368dc)
                 |     4     |
                 -------------  (name.ptr: 1043368e4)
                 | 1182145c0 |-------------------------
levelUp: user -> -------------  (id: 1043368ec)       |
                 |     1     |                        |
                 -------------  (power: 1043368f4)    |
                 |    100    |                        |
                 -------------  (name.len: 1043368f8) |
                 |     4     |                        |
                 -------------  (name.ptr: 104336900) |
                 | 1182145c0 |-------------------------
                 -------------                        |
                                                      |
                 .............  espaço vazio          |
                 .............  ou outros dados       |
                                                      |
                 -------------  (1182145c0)        <---
                 |    'G'    |
                 -------------
                 |    'o'    |
                 -------------
                 |    'k'    |
                 -------------
                 |    'u'    |
                 -------------
```

A partir do que foi apresentado, podemos ver que a cópia rasa funcionará. Como o valor de um ponteiro é um endereço, copiar o valor significa que obtemos o mesmo endereço. Isso tem implicações importantes em relação à mutabilidade. Nossa função não pode alterar os campos diretamente acessíveis por `main.user`, já que ela obteve uma cópia, mas ela tem acesso ao mesmo `name`, então pode modificá-lo? Neste caso específico, não, `name` é uma constante (`const`). Além disso, nosso valor "Goku" é uma string literal, que é sempre imutável. No entanto, com um pouco de trabalho, podemos ver a implicação da cópia rasa:

```zig
const std = @import("std");

pub fn main() void {
	var name = [4]u8{'G', 'o', 'k', 'u'};
	var user = User{
		.id = 1,
		.power = 100,
		// transformando num slice, [4]u8 -> []u8
		.name = name[0..],
	};
	levelUp(user);
	std.debug.print("{s}\n", .{user.name});
}

fn levelUp(user: User) void {
	user.name[2] = '!';
}

pub const User = struct {
	id: u64,
	power: i32,
	// []const u8 -> []u8
	name: []u8
};
```

O código acima imprime "Go!u". Tivemos que mudar o tipo de `name` de `[]const u8` para `[]u8` e, em vez de uma string literal, que é sempre imutável, criar uma matriz e fatiá-la. Alguns podem ver inconsistência aqui. Passar por valor impede que uma função mute campos imediatos, mas não campos com um valor por trás de um ponteiro. Se quiséssemos que `name` fosse imutável, deveríamos tê-lo declarado como `[]const u8` em vez de `[]u8`.

Algumas linguagens têm uma implementação diferente, mas muitas linguagens funcionam exatamente assim (ou muito parecido). Embora tudo isso possa parecer esotérico, é fundamental para a programação diária. A boa notícia é que você pode dominar isso usando exemplos e trechos simples; não fica mais complicado à medida que outras partes do sistema crescem em complexidade.



## Estruturas recursivas

Às vezes, você precisa que uma estrutura seja recursiva. Mantendo nosso código existente, vamos adicionar uma varíavel `manager` opcional do tipo `?User` ao nosso `User`. Enquanto estamos nisso, criaremos dois `Users` e atribuiremos um como gerente de outro:

```zig
const std = @import("std");

pub fn main() void {
	const leto = User{
		.id = 1,
		.power = 9001,
		.manager = null,
	};

	const duncan = User{
		.id = 1,
		.power = 9001,
		.manager = leto,
	};

	std.debug.print("{any}\n{any}", .{leto, duncan});
}

pub const User = struct {
	id: u64,
	power: i32,
	manager: ?User,
};
```

Este código não vai compilar: _struct 'learning.User' depends on itself_ (dependência de si própria). Isso falha porque todo tipo precisa ter um tamanho conhecido em tempo de compilação.

Não tivemos esse problema ao adicionar `name`, mesmo que os nomes possam ter comprimentos diferentes. O problema não é com o tamanho dos valores, mas com o tamanho dos tipos em si. Zig precisa dessa informação para fazer tudo o que discutimos acima, como acessar um campo com base na posição do seu deslocamento. `name` era uma fatia, um `[]const u8`, e isso tem um tamanho conhecido: 16 bytes - 8 bytes para `len` e 8 bytes para `ptr`.

Você pode pensar que isso será um problema com qualquer opcional ou união. Mas para ambos os opcionais e uniões, o tamanho máximo possível é conhecido e Zig pode usá-lo. Uma estrutura recursiva não tem tal limite superior, a estrutura pode se repetir uma vez, duas vezes ou milhões de vezes. Esse número variaria de `User` para `User` e não seria conhecido em tempo de compilação.

Vimos a resposta com `name`: use um ponteiro. Ponteiros sempre ocupam `usize` bytes. Em uma plataforma de 64 bits, isso são 8 bytes. Assim como o nome real "Goku" não foi armazenado com/ao lado do nosso `user`, usar um ponteiro significa que nosso gerente não está mais vinculado ao layout de memória de `user`.

```zig
const std = @import("std");

pub fn main() void {
	const leto = User{
		.id = 1,
		.power = 9001,
		.manager = null,
	};

	const duncan = User{
		.id = 1,
		.power = 9001,
		// mudança: leto -> &leto
		.manager = &leto,
	};

	std.debug.print("{any}\n{any}", .{leto, duncan});
}

pub const User = struct {
	id: u64,
	power: i32,
	// mudança: ?const User -> ?*const User
	manager: ?*const User,
};
```

Você pode nunca precisar de uma estrutura recursiva, mas isso não se trata de modelagem de dados. Trata-se de entender ponteiros e modelos de memória e compreender melhor o que o compilador está fazendo.

---

Muitos desenvolvedores têm dificuldade com ponteiros, pode haver algo evasivo sobre eles. Eles não parecem concretos como um número inteiro, ou uma string ou um `User`. Nada disso precisa estar totalmente claro para você avançar. Mas vale a pena dominar, e não apenas para o Zig. Esses detalhes podem estar ocultos em linguagens como Ruby, Python e JavaScript, e em menor medida em C#, Java e Go, mas ainda estão lá, impactando como você escreve código e como esse código é executado. Então, vá com calma, experimente exemplos, adicione declarações de impressão de depuração para examinar variáveis e seus endereços. Quanto mais você explorar, mais claro ficará.
