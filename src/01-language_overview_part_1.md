# Visão Geral da Linguagem - Parte 1

Zig é uma linguagem compilada fortemente tipada. Ela suporta genéricos (parametrização polimórfica), possui poderosas capacidades de metaprogramação em tempo de compilação e **não inclui** um coletor de lixo. Muitas pessoas consideram o Zig uma alternativa moderna ao C. Como tal, a sintaxe da linguagem é semelhante à do C. Estamos falando de declarações terminadas por ponto e vírgula e blocos delimitados por chaves.

Aqui está como se parece o código Zig:

```zig
const std = @import("std");

// Este código não irá compilar caso a função `main` não seja `pub` (tenha visibilidade pública)
pub fn main() void {
	const user = User{
		.power = 9001,
		.name = "Goku",
	};

	std.debug.print("{s}'s power is {d}\n", .{user.name, user.power});
}

pub const User = struct {
	power: u64,
	name: []const u8,
};
```

Se você salvar o código acima como *learning.zig* e executar `zig run learning.zig`, você deverá ver: `Goku's power is 9001`.

Este é um exemplo simples, algo que você pode conseguir acompanhar mesmo que seja a primeira vez que você está vendo um codigo em Zig. Ainda assim, vamos analisar linha por linha.

> Consulte a seção de [instalação do Zig](https://www.openmymind.net/learning_zig/#install) para configurar rapidamente e começar a usar.



## Importação

Muito poucos programas são escritos como um único arquivo sem uma biblioteca padrão ou bibliotecas externas. Nosso primeiro programa não é exceção e utiliza a biblioteca padrão do Zig para imprimir nossa saída. O sistema de importação do Zig é direto e depende da função `@import` e da palavra-chave `pub` (para tornar o código acessível fora do arquivo atual).

> Funções que começam com `@` são funções integradas (nativas em nível de compilador). Estas são fornecidas pelo compilador, constrastando com aquelas fornecidas pela biblioteca padrão.

Importamos um módulo especificando o nome do módulo. A biblioteca padrão do Zig está disponível usando o nome "std". Para importar um arquivo específico, utilizamos o seu caminho relativo ao arquivo que está fazendo a importação. Por exemplo, se movermos a estrutura `User` para seu próprio arquivo, digamos _models/user.zig_:

```zig
// models/user.zig
pub const User = struct {
	power: u64,
	name: []const u8,
};
```

Então, o importaríamos da seguinte forma:

```zig
// main.zig
const User = @import("models/user.zig").User;
```

> Se nosso _struct_ `User` não estiver marcada como `pub`, receberemos o seguinte erro: _'User' não está marcada como 'pub'_.

_models/user.zig_ pode exportar mais de uma coisa. Por exemplo, também poderíamos exportar uma constante:

```zig
// models/user.zig
pub const MAX_POWER = 100_000;

pub const User = struct {
	power: u64,
	name: []const u8,
};
```

Neste caso, poderíamos importar ambos:

```zig
const user = @import("models/user.zig");
const User = user.User;
const MAX_POWER = user.MAX_POWER
```

Neste ponto, você pode ter mais perguntas do que respostas. No trecho acima, o que é `user`? Ainda não o vimos, mas e se usarmos `var` em vez de `const`? Ou talvez você esteja se perguntando como usar bibliotecas de terceiros. São todas boas perguntas, mas para respondê-las, primeiro precisamos aprender mais sobre Zig. Por enquanto, teremos que ficar satisfeitos com o que aprendemos: como importar a biblioteca padrão do Zig, como importar outros arquivos e como exportar definições.



## Comentários

A próxima linha no nosso exemplo Zig é um comentário:

```zig
// Este código não irá compilar caso a função `main` não seja `pub` (tenha visibilidade pública)
```

O Zig não tem comentários de várias linhas, como os `/* ... */` em C.

Existe suporte experimental para geração automatizada de documentação com base em comentários. Se você já viu a [documentação da biblioteca padrão do Zig](https://ziglang.org/documentation/master/std), então você já viu isso em ação. `//!` é conhecido como um comentário de documento de nível superior e pode ser colocado no início do arquivo. Um comentário de três barras (`///`), conhecido como comentário de documento, pode ser colocado em lugares específicos, como antes de uma declaração. Você receberá um erro do compilador se tentar usar qualquer tipo de comentário de documento no lugar errado.



## Funções

Nossa próxima linha de código é o início da nossa função principal (`main`):

```zig
pub fn main() void
```

Todo executável precisa de uma função chamada `main`: é o ponto de entrada do programa. Se renomeássemos `main` para algo diferente, como `doIt`, e tentássemos executar `zig run learning.zig`, receberíamos um erro: _'learning' has no member named 'main'_ (dizendo que _'learning'_ não tem um membro chamado _'main'_).

Ignorando o papel especial de `main` como o ponto de entrada do nosso programa, é uma função bastante básica: não recebe parâmetros e não retorna nada, ou seja, `void`. O seguinte é _um pouco_ mais interessante:

```zig
const std = @import("std");

pub fn main() void {
	const sum = add(8999, 2);
	std.debug.print("8999 + 2 = {d}\n", .{sum});
}

fn add(a: i64, b: i64) i64 {
	return a + b;
}
```

Programadores de C e C++ perceberão que em Zig não se exige declarações pré-definidas, ou seja, a função `add` é chamada _antes_ de ser definida.

A próxima coisa a notar é o tipo `i64`: um inteiro de 64 bits com marcação. Alguns outros tipos numéricos são: `u8`, `i8`, `u16`, `i16`, `u32`, `i32`, `u47`, `i47`, `u64`, `i64`, `f32` e `f64`. A inclusão de `u47` e `i47` não é um teste para garantir que você ainda está acordado; Zig suporta inteiros de tamanho arbitrário em bits. Embora você provavelmente não os use com frequência, eles podem ser úteis. Um tipo que você usará com frequência é `usize`, que é um inteiro sem marcação do tamanho de um ponteiro e geralmente o tipo que representa o comprimento/tamanho de algo.

> Além de `f32` e `f64`, Zig também suporta os tipos de ponto flutuante `f16`, `f80` e `f128`.

Embora não haja uma boa razão para fazer isso, se mudarmos a implementação de `add` para:

```zig
fn add(a: i64, b: i64) i64 {
	a += b;
	return a;
}
```

Vamos obter um erro em `a += b;`: _cannot assign to constant_ (não é possível atribuir a uma constante). Esta é uma lição importante que revisaremos com mais detalhes posteriormente: os parâmetros de função são constantes.

Para melhorar a legibilidade, não há sobrecarga de funções (a mesma função nomeada definida com tipos de parâmetros e/ou número de parâmetros diferentes). Por enquanto, isso é tudo o que precisamos saber sobre funções.



## Estruturas _(struct)_

A próxima linha de código é a criação de um `User`, um tipo que é definido no final do nosso trecho. A definição de `User` é:

```zig
pub const User = struct {
	power: u64,
	name: []const u8,
};
```

> Como nosso programa é um único arquivo e, portanto, `User` é usado apenas no arquivo onde é definido, não precisávamos torná-lo `pub`. Mas, então, não teríamos visto como expor uma declaração para outros arquivos.

Os campos de um `struct` são terminados com uma vírgula e podem ser atribuídos um valor padrão:

```zig
pub const User = struct {
	power: u64 = 0,
	name: []const u8,
};
```

Quando criamos um `struct`, **cada campo** deve ser definido. Por exemplo, na definição original, onde `power` não tinha um valor padrão, o seguinte geraria um erro: _missing struct field: power_.

```zig
const user = User{.name = "Goku"};
```

No entanto, com o nosso valor padrão, o código acima compila normalmente.

_Structs_ podem ter métodos, podem conter declarações (incluindo outros _structs_) e até mesmo podem não ter nenhum campo, nesse caso agindo mais como um _namespace_.

```zig
pub const User = struct {
	power: u64 = 0,
	name: []const u8,

	pub const SUPER_POWER = 9000;

	fn diagnose(user: User) void {
		if (user.power >= SUPER_POWER) {
			std.debug.print("it's over {d}!!!", .{SUPER_POWER});
		}
	}
};
```

Métodos são apenas funções normais que podem ser chamadas com uma sintaxe de ponto. Ambas funções funcionam:

```zig
// Chamar a função através da variável
user.diagnose();

// É o mesmo que chamar a função através do tipo, passando a variável como argumento
User.diagnose(user);
```

Na maioria das vezes, você usará a sintaxe de ponto, mas os métodos como uma espécie simplificação na sintaxe sobre funções normais podem ser úteis.

> A instrução `if` é o primeiro controle de fluxo que vimos. É bastante direta, não é? Vamos explorar isso com mais detalhes na próxima parte.

A função `diagnose` é definida dentro do nosso tipo `User` e aceita um `User` como seu primeiro parâmetro. Como tal, podemos chamá-la com a sintaxe de ponto. Mas as funções dentro de uma estrutura não precisam seguir esse padrão. Um exemplo comum é ter uma função `init` para iniciar nossa estrutura:

```zig
pub const User = struct {
	power: u64 = 0,
	name: []const u8,

	pub fn init(name: []const u8, power: u64) User {
		return User{
			.name = name,
			.power = power,
		};
	}
}
```

O uso de `init` é apenas uma convenção e, em alguns casos, `open` ou algum outro nome pode fazer mais sentido. Se você, como eu, não é um(a) programador(a) de C++, a sintaxe para inicializar campos, `.$campo = $valor`, pode parecer um pouco estranha, mas você se acostumará rapidamente.

Quando criamos "Goku", declaramos a variável `user` como `const`:

```zig
const user = User{
	.power = 9001,
	.name = "Goku",
};
```

Isso significa que não podemos modificar `user`. Para modificar uma variável, ela deve ser declarada usando `var`. Além disso, você pode ter notado que o tipo de `user` é inferido com base no que é atribuído a ele. Poderíamos ser explícitos:

```zig
const user: User = User{
	.power = 9001,
	.name = "Goku",
};
```

Veremos casos em que precisamos ser explícitos sobre o tipo de uma variável, mas na maioria das vezes, o código é mais legível sem o tipo explícito. A inferência de tipo funciona da mesma forma. Isso é equivalente a ambos os trechos acima:

```zig
const user: User = .{
	.power = 9001,
	.name = "Goku",
};
```

No entanto, essa forma de uso é bastante incomum. Um lugar onde é mais comum é ao retornar uma estrutura de uma função. Aqui, o tipo pode ser inferido a partir do tipo de retorno da função. Nossa função `init` provavelmente seria escrita assim:

```zig
pub fn init(name: []const u8, power: u64) User {
	// Ao invés de retornar "User{...}"
	return .{
		.name = name,
		.power = power,
	};
}
```

Como a maioria das coisas que exploramos até agora, revisaremos `structs` no futuro quando falarmos sobre outras partes da linguagem. Mas, na maior parte, são bem simples.



## Vetores _(arrays)_ e Segmentos _(slices)_

Podemos pular a última linha do nosso código, mas dado que nosso pequeno trecho contém duas _strings_ (cadeia de caraceteres), **"Goku"** e **"{s}'s power is {d}\n"**, você provavelmente está curioso sobre como funcionam as _strings_ em Zig. Para entender melhor as _strings_, vamos primeiro explorar _arrays_ e _slices_.

_Arrays_ possuem tamanho fixo com um comprimento conhecido em tempo de compilação. O comprimento faz parte do tipo, portanto, um _array_ de 4 inteiros assinados, `[4]i32`, é um tipo diferente de um _array_ de 5 inteiros assinados, `[5]i32`.

O comprimento do _array_ pode ser inferido a partir da inicialização. No código a seguir, todas as três variáveis são do tipo `[5]i32`:

```zig
const a = [5]i32{1, 2, 3, 4, 5};

// nós já vimos esta .{...} sintaxe com structs
// ela também funciona com arrays
const b: [5]i32 = .{1, 2, 3, 4, 5};

// use a notação _ para deixar o compilador inferir o comprimento do array
const c = [_]i32{1, 2, 3, 4, 5};
```

Um slice (segmento), por outro lado, é um ponteiro para um array com um comprimento. O comprimento é conhecido em tempo de execução. Abordaremos ponteiros em uma parte posterior, mas você pode pensar em um slice como uma visão (segmento) de parte do array.

> Se você está familiarizado com Go, pode ter notado que as slices em Zig são um pouco diferentes: elas não têm uma capacidade, apenas um ponteiro e um comprimento.

Dado o seguinte bloco de código,

```zig
const a = [_]i32{1, 2, 3, 4, 5};
const b = a[1..4];
```

Eu adoraria poder te dizer que `b` é um slice com um comprimento de 3 e um ponteiro para `a`. No entanto, porque "fatiamos" nosso array usando valores conhecidos em tempo de compilação, ou seja, `1` e `4`, nosso comprimento, `3`, também é conhecido em tempo de compilação. O compilador do Zig analisa tudo isso e, portanto, `b` não é realmente um slice, mas sim um ponteiro para um array de inteiros com um comprimento de 3. Especificamente, seu tipo é `*const [3]i32`. Portanto, esta demonstração de um slice é frustrada pela engenhosidade e capacidade de análise do compilador do Zig.

Em uma bse de código real, você provavelmente usará mais slices do que arrays. Para o bem ou para o mal, os programas tendem a ter mais informações em tempo de execução do que em tempo de compilação. Em um exemplo pequeno, no entanto, precisamos enganar o compilador para obter o que queremos:

```zig
const a = [_]i32{1, 2, 3, 4, 5};
var end: usize = 4;
const b = a[1..end];
```

Agora, `b` é, de fato, um slice. Mais especificamente, seu tipo é `[]const i32`. Você pode perceber que o comprimento do slice não faz parte do tipo, porque o comprimento é uma propriedade em tempo de execução, e os tipos são sempre totalmente conhecidos em tempo de compilação. Ao criar um slice, podemos omitir o limite superior para criar um slice até o final do que estamos fatiando (seja um array ou um slice), por exemplo, `const c = b[2..];`.

> Se tivéssemos declarado `end` como `const`, ela teria se tornado um valor conhecido em tempo de compilação, o que teria resultado em um comprimento conhecido em tempo de compilação para `b` e, portanto, criado um ponteiro para um array, e não um slice. Eu acho isso um pouco confuso, mas não é algo que surge com muita frequência e não é muito difícil de dominar. Eu adoraria pular isso neste momento, mas não consegui encontrar uma maneira legítima de evitar esse detalhe.

Aprender a linguagem Zig me ensinou que os tipos são muito descritivos. Não é apenas um número inteiro ou um booleano, ou mesmo um array de inteiros de 32 bits com sinal. Os tipos também contêm outras informações importantes. Já falamos sobre o comprimento fazer parte do tipo de um array, e muitos dos exemplos mostraram como a constância também faz parte dele. Por exemplo, em nosso último exemplo, o tipo de `b` é `[]const i32`. Você pode verificar isso por si mesmo com o seguinte código:

```zig
const std = @import("std");

pub fn main() void {
	const a = [_]i32{1, 2, 3, 4, 5};
	var end: usize = 4;
	const b = a[1..end];
	std.debug.print("{any}", .{@TypeOf(b)});
}
```

Se tentássemos escrever em `b`, como por exemplo `b[2] = 5;`, obteríamos um erro de compilação: _"cannot assign to constant"_ (não é possível atribuir a uma constante). Isso ocorre devido ao tipo de `b`.

Para resolver isso, você pode ser tentado a fazer a seguinte alteração:

```zig
// substituição de const por var
var b = a[1..end];
```

Mas você obterá o mesmo erro, por quê? Como dica, fica a questão: qual é o tipo de `b`, ou mais genericamente, o que é `b`? Um slice é um comprimento e um ponteiro para **parte de** um array. O tipo de um slice é sempre derivado do array subjacente. Se `b` for declarado como `const` ou não, o array subjacente é do tipo `[5]const i32`, e assim `b` deve ser do tipo `[]const i32`. Se quisermos poder escrever em `b`, precisamos alterar `a` de `const` para `var`.

```zig
const std = @import("std");

pub fn main() void {
	var a = [_]i32{1, 2, 3, 4, 5};
	var end: usize = 4;
	const b = a[1..end];
	b[2] = 99;
}
```

Isso funciona porque nosso slice não é mais `[]const i32`, mas sim `[]i32`. Você pode estar se perguntando por que isso funciona quando `b` ainda é `const` (constante). Mas a constância de `b` está relacionada a `b` em si, não aos dados que `b` aponta. Bem, não tenho certeza se essa é uma ótima explicação, mas para mim, este código destaca a diferença:

```zig
const std = @import("std");

pub fn main() void {
	var a = [_]i32{1, 2, 3, 4, 5};
	var end: usize = 4;
	const b = a[1..end];
	b = b[1..];
}
```

Este código não irá compilar; como o compilador nos informa: _"cannot assign to constant"_ (não podemos atribuir a uma constante). Mas se tivéssemos feito `var b = a[1..end];`, então o código teria funcionado porque `b` em si já não é uma constante.

Vamos descobrir mais sobre arrays e slices enquanto exploramos outros aspectos da linguagem, incluindo strings, que não são menos importantes.



## Cadeia de caracteres _(strings)_

Eu gostaria de poder dizer que Zig possui um tipo de `string` e que é incrível. Infelizmente, não é o caso. Em sua forma mais simples, as strings em Zig são sequências (ou seja, arrays ou slices) de bytes (`u8`). Vimos isso na definição do campo `name`: `name: []const u8`.

Por convenção, e apenas por convenção, essas strings devem conter apenas valores UTF-8, já que o código-fonte Zig é ele mesmo codificado em UTF-8. No entanto, isso não é imposto pelo compilador, e não há realmente nenhuma diferença entre um `[]const u8` que representa uma string ASCII ou UTF-8 e um `[]const u8` que representa dados binários arbitrários. Como poderia haver, eles são do mesmo tipo.

Com base no que aprendemos sobre arrays e slices, você estaria correto ao supor que `[]const u8` é uma slice de um array constante de bytes (onde um byte é um inteiro sem sinal de 8 bits). Mas em nenhum lugar do nosso código fatiamos um array ou mesmo tivemos um array, certo? Tudo o que fizemos foi atribuir "Goku" a `user.name`. Como isso funcionou?

As literais de string, aquelas que você vê no código-fonte, têm um comprimento conhecido em tempo de compilação. O compilador sabe que "Goku" tem um comprimento de 4. Então, você estaria próximo ao pensar que "Goku" é melhor representado por um array, algo como `[4]const u8`. Mas as literais de string têm algumas propriedades especiais. Elas são armazenadas em um local especial dentro do binário e são deduplicadas. Portanto, uma variável para uma literal de string será um ponteiro para este local especial. Isso significa que o tipo de "Goku" está mais próximo de `*const [4]u8`, um ponteiro para um array constante de 4 bytes.

Tem mais. As literais de string são terminadas por um caractere nulo. Ou seja, elas sempre têm um `\0` no final. Strings terminadas por nulo são importantes ao interagir com C. Na memória, "Goku" realmente se pareceria com: `{'G', 'o', 'k', 'u', 0}`, então você poderia pensar que o tipo é `*const [5]u8`. Mas isso seria ambíguo na melhor das hipóteses e perigoso na pior das hipóteses (você poderia sobrescrever o terminador nulo). Em vez disso, Zig tem uma sintaxe distinta para representar arrays terminados por nulo. "Goku" tem o tipo: `*const [4:0]u8`, um ponteiro para um array de 4 bytes terminado por nulo. Embora estejamos falando sobre strings, estamos nos concentrando em arrays de bytes terminados por nulo (já que é assim que as strings são tipicamente representadas em C), a sintaxe é mais genérica: `[COMPRIMENTO:MARCADOR]`, onde "MARCADOR" é o valor especial encontrado no final do array. Então, embora eu não consiga pensar em um motivo para isso, o seguinte é completamente válido:

```zig
const std = @import("std");

pub fn main() void {
    // um array de 3 booleanos com false sendo o marcador
	const a = [3:false]bool{false, true, false};

	// Esta linha de código é mais avançada e não será explicada!
	std.debug.print("{any}\n", .{std.mem.asBytes(&a).*});
}
```

Cuja saída será: `{ 0, 1, 0, 0}`.

> Eu hesitei em incluir este exemplo, já que a última linha é bastante avançada e eu não pretendo explicá-la. Por outro lado, é um exemplo funcional que você pode executar e experimentar para examinar melhor alguns dos conceitos que discutimos até agora, se você estiver interessado.

Se eu consegui explicar isso de forma aceitável, provavelmente ainda há uma coisa da qual você está incerto. Se "Goku" é um `*const [4:0]u8`, como conseguimos atribuí-lo a `name`, um `[]const u8`? A resposta é simples: o Zig fará a coerção de tipo para você. Ele fará isso entre alguns tipos diferentes, mas é mais óbvio com strings. Isso significa que se uma função tiver um parâmetro `[]const u8`, ou uma estrutura tiver um campo `[]const u8`, literais de string podem ser usadas. Como as strings terminadas por nulo são arrays, e os arrays têm um comprimento conhecido, essa coerção é barata, ou seja, **não** requer a iteração pela string para encontrar o terminador nulo.

Portanto, ao falar sobre strings, geralmente nos referimos a um `[]const u8`. Quando necessário, explicitamente mencionamos uma string terminada por nulo, que pode ser automaticamente coercida para um `[]const u8`. Mas lembre-se de que um `[]const u8` também é usado para representar dados binários arbitrários, e, como tal, o Zig não tem a noção de uma string como as linguagens de programação de nível mais alto têm. Além disso, a biblioteca padrão do Zig possui apenas um módulo unicode muito básico.

Claro, em um programa real, a maioria das strings (e de forma mais genérica, arrays) não são conhecidas em tempo de compilação. O exemplo clássico é a entrada do usuário, que não é conhecida quando o programa está sendo compilado. Isso é algo que teremos que revisitar ao falar sobre memória. Mas a resposta curta é que, para esses dados, que têm um valor desconhecido em tempo de compilação e, portanto, um comprimento desconhecido, alocaremos dinamicamente memória em tempo de execução. Nossas variáveis de string, ainda do tipo `[]const u8`, serão slices que apontam para essa memória alocada dinamicamente.



## Palavras-chave _comptime_ e _anytype_

Há muito mais acontecendo na última linha de código não explorada:

```zig
std.debug.print("{s}'s power is {d}\n", .{user.name, user.power});
```

Vamos apenas dar uma olhada superficial, mas isso oferece a oportunidade de destacar alguns dos recursos mais avançados do Zig. Essas são coisas das quais você pelo menos deve estar ciente, mesmo que não as tenha dominado.

O primeiro é o conceito em Zig de execução em tempo de compilação, ou `comptime`. Isso é fundamental para as capacidades de metaprogramação do Zig e, como o nome sugere, envolve a execução de código em tempo de compilação, em vez de tempo de execução. Ao longo deste guia, apenas arranharemos a superfície do que é possível com `comptime`, mas é algo que está sempre presente.

Você pode estar se perguntando o que há na linha acima que exige a execução em tempo de compilação. A definição da função `print` exige que nosso primeiro parâmetro, o formato da string, seja conhecido em tempo de compilação:

```zig
// perceba "comptime" antes da variável "fmt"
pub fn print(comptime fmt: []const u8, args: anytype) void {
```

E a razão para isso é que a função `print` realiza verificações extras em tempo de compilação que você não teria na maioria das outras linguagens. Que tipo de verificações? Bem, digamos que você altere o formato para `"it's over {d}\n"`, mas mantenha os dois argumentos. Você receberá um erro de compilação: _unused argument in 'it's over {d}'_ (argumento não utilizado em _'it's over {d}'_). Ela também faz verificações de tipo: altere a string de formato para `"{s}'s power is {s}\n"` e você receberá: _invalid format string 's' for type 'u64'_ (string de formato inválida 's' para o tipo 'u64'). Essas verificações não seriam possíveis de serem feitas em tempo de compilação se o formato da string não fosse conhecido em tempo de compilação. Daí a necessidade de um valor conhecido em tempo de compilação.

O único lugar onde o `comptime` impactará imediatamente o seu código são os tipos padrão para literais de inteiros e ponto flutuante, os tipos especiais `comptime_int` e `comptime_float`. Esta linha de código não é válida: `var i = 0;`. Você receberá um erro de compilação: _variable of type 'comptime_int' must be const or comptime_ (a variável do tipo 'comptime_int' deve ser const ou comptime). O código `comptime` só pode trabalhar com dados que são conhecidos em tempo de compilação e, para inteiros e pontos-flutuantes, esses dados são identificados pelos tipos especiais `comptime_int` e `comptime_float`. Um valor desse tipo pode ser usado na execução em tempo de compilação. Mas provavelmente você não passará a maior parte do seu tempo escrevendo código para a execução em tempo de compilação, então isso não é particularmente útil por padrão. O que você precisará fazer é dar um tipo explícito às suas variáveis:

```zig
var i: usize = 0;
var j: f64 = 0;
```

> Observe que esse erro ocorreu apenas porque usamos `var`. Se tivéssemos usado `const`, não teríamos recebido o erro, já que o ponto central do erro é que um `comptime_int` __deve__ ser constante.

Em uma parte futura, examinaremos o comptime um pouco mais ao explorar os genéricos.

A outra coisa especial sobre nossa linha de código é o estranho `. {user.name, user.power}`, que, a partir da definição de `print` acima, sabemos que mapeia para uma variável do tipo `anytype`. Esse tipo não deve ser confundido com algo como o `Object` em Java ou o `any` (também conhecido como `interface{}`) em Go. Em vez disso, em tempo de compilação, o Zig criará uma versão da função `print` especificamente para todos os tipos que foram passados para ela.

Isso nos leva à pergunta: o que estamos passando para ela? Já vimos a notação `.{ ... }` antes, ao permitir que o compilador infira o tipo da nossa estrutura. Isso é semelhante: cria um literal de estrutura anônima. Considere este código:

```zig
pub fn main() void {
	std.debug.print("{any}\n", .{@TypeOf(.{.year = 2023, .month = 8})});
}
```

cuja saída no terminal será:

```zig
struct{comptime year: comptime_int = 2023, comptime month: comptime_int = 8}
```

Aqui, demos nomes aos campos de nossa estrutura anônima, `year` e `month`. Em nosso código original, não o fizemos. Nesse caso, os nomes dos campos são gerados automaticamente como "0", "1", "2", etc. A função `print` espera uma estrutura com esses campos e usa a posição ordinal na string de formato para obter o argumento apropriado.

Zig não possui sobrecarga de funções, e não possui funções variádicas (funções com um número arbitrário de argumentos). Mas ele tem um compilador capaz de criar funções especializadas com base nos tipos passados, incluindo tipos inferidos e criados pelo próprio compilador.
