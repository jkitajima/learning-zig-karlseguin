# Memória Dinâmica & Alocadores

Tudo o que vimos até agora foi restrito pela necessidade de um tamanho definido antecipadamente. Arrays sempre têm um comprimento conhecido em tempo de compilação (na verdade, o comprimento faz parte do tipo). Todas as nossas strings foram literais de string, que têm um comprimento conhecido em tempo de compilação.

Além disso, as duas estratégias de gerenciamento de memória que vimos, dados globais e a pilha de chamadas, embora simples e eficientes, são limitadas. Nenhuma delas pode lidar com dados de tamanho dinâmico e ambas são rígidas em relação aos períodos de vida dos dados.

Esta parte está dividida em dois temas. O primeiro é uma visão geral do nosso terceiro espaço de memória, o heap. O outro é a abordagem direta, mas única, do Zig para o gerenciamento de memória no heap. Mesmo que você esteja familiarizado com a memória do heap, digamos, ao usar o `malloc` do C, você vai querer ler a primeira parte, pois ela é bastante específica para o Zig.



## Memória dinâmica _(heap)_

O heap é a terceira e última área de memória à nossa disposição. Em comparação com os dados globais e a pilha de chamadas, o heap é um pouco como o "oeste selvagem": tudo é permitido. Especificamente, dentro do heap, podemos criar memória em tempo de execução com um tamanho conhecido em tempo de execução e ter controle total sobre o tempo de vida dessa memória.

A pilha de chamadas é incrível devido à maneira simples e previsível como ela gerencia dados (empilhando e desempilhando frames de pilha). Esse benefício também é uma desvantagem: os dados têm uma vida útil vinculada ao seu lugar na pilha de chamadas. O heap é exatamente o oposto. Ele não possui um ciclo de vida embutido, então nossos dados podem viver pelo tempo que for necessário. E esse benefício é sua desvantagem: ele não possui um ciclo de vida embutido, então se não liberarmos a memória, ninguém o fará.

Vamos ver um exemplo:

```zig
const std = @import("std");

pub fn main() !void {
    // falaremos sobre alocadores na sequência
	var gpa = std.heap.GeneralPurposeAllocator(.{}){};
	const allocator = gpa.allocator();

	// ** as próximas duas linhas são as importantes **
	var arr = try allocator.alloc(usize, try getRandomCount());
	defer allocator.free(arr);

	for (0..arr.len) |i| {
		arr[i] = i;
	}
	std.debug.print("{any}\n", .{arr});
}

fn getRandomCount() !u8 {
	var seed: u64 = undefined;
	try std.os.getrandom(std.mem.asBytes(&seed));
	var random = std.rand.DefaultPrng.init(seed);
	return random.random().uintAtMost(u8, 5) + 5;
}
```

Em breve abordaremos os alocadores do Zig; por enquanto, saiba que o `allocator` é um `std.mem.Allocator`. Estamos utilizando dois de seus métodos: `alloc` e `free`. Porque estamos chamando `allocator.alloc` com um `try`, sabemos que pode falhar. Atualmente, o único erro possível é `OutOfMemory`. Seus parâmetros em sua maioria nos dizem como ele funciona: ele quer um tipo (`T`) e um contador e, em caso de sucesso, retorna uma fatia de `[]T`. Essa alocação acontece em tempo de execução - precisa ser assim, nosso contador só é conhecido em tempo de execução.

Como regra geral, cada `alloc` terá um `free` correspondente. Enquanto `alloc` aloca memória, `free` a libera. Não deixe este código simples limitar sua imaginação. Este padrão comum de `try alloc` + `defer free` é utilizado com frequência, e por uma boa razão: liberar próximo ao local da alocação é relativamente à prova de falhas. Mas igualmente comum é alocar em um lugar enquanto libera em outro. Como mencionamos anteriormente, o heap não possui gerenciamento de ciclo de vida embutido. Você pode alocar memória em um manipulador de HTTP e liberá-la em uma thread de segundo plano, duas partes completamente separadas do código.



## defer & errdefer

Em uma pequena digressão, o código acima introduziu um novo recurso da linguagem: `defer`, que executa o código ou bloco fornecido na saída do escopo. "Saída do escopo" inclui alcançar o final do escopo ou retornar do escopo. `defer` não está estritamente relacionado a alocadores ou gerenciamento de memória; você pode usá-lo para executar qualquer código. Mas o uso acima é comum.

O _defer_ do Zig é semelhante ao do Go, com uma diferença importante. No Zig, o _defer_ será executado no final de seu escopo contêiner. No Go, o _defer_ é executado no final da função contêiner. A abordagem do Zig provavelmente é menos surpreendente, a menos que você seja um desenvolvedor Go.

Um parente do `defer` é o `errdefer`, que de forma semelhante executa o código ou bloco fornecido na saída do escopo, mas apenas quando um erro é retornado. Isso é útil ao fazer configurações mais complexas e ter que desfazer uma alocação anterior por causa de um erro.

O exemplo a seguir é um salto em complexidade. Ele apresenta tanto o `errdefer` quanto um padrão comum que envolve alocar em `init` e liberar em `deinit`:

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

pub const Game = struct {
	players: []Player,
	history: []Move,
	allocator: Allocator,

	fn init(allocator: Allocator, player_count: usize) !Game {
		var players = try allocator.alloc(Player, player_count);
		errdefer allocator.free(players);

		// armazena as 10 movimentações mais recentes por jogador
		var history = try allocator.alloc(Move, player_count * 10);

		return .{
			.players = players,
			.history = history,
			.allocator = allocator,
		};
	}

	fn deinit(game: Game) void {
		const allocator = game.allocator;
		allocator.free(game.players);
		allocator.free(game.history);
	}
};
```

Espero que isso destaque duas coisas. Primeiro, a utilidade do `errdefer`. Sob condições normais, `players` é alocado em `init` e liberado em `deinit`. Mas há um caso limite quando a inicialização de `history` falha. Neste caso e apenas neste caso, precisamos desfazer a alocação de `players`.

O segundo aspecto digno de nota neste código é que o ciclo de vida de nossas duas fatias alocadas dinamicamente, `players` e `history`, é baseado na lógica de nossa aplicação. Não há uma regra que dite quando `deinit` deve ser chamado ou quem deve chamá-lo. Isso é bom, porque nos dá tempos de vida arbitrários, mas é ruim porque podemos estragar as coisas ao nunca chamar `deinit` ou chamá-lo mais de uma vez.

> Os nomes `init` e `deinit` não são especiais. Eles são apenas o que a biblioteca padrão do Zig usa e o que a comunidade adotou. Em alguns casos, incluindo na biblioteca padrão, são usados `open` e `close`, ou outros nomes mais apropriados.



## Liberação dupla _(double free)_ & Vazamento de memória _(memory leak)_

Logo acima, mencionei que não há regras que determinam quando algo deve ser liberado. Mas isso não é totalmente verdade; há algumas regras importantes, apenas não são impostas, exceto pela sua própria meticulosidade.

A primeira regra é que você não pode liberar a mesma memória duas vezes.

```zig
const std = @import("std");

pub fn main() !void {
	var gpa = std.heap.GeneralPurposeAllocator(.{}){};
	const allocator = gpa.allocator();

	var arr = try allocator.alloc(usize, 4);
	allocator.free(arr);
	allocator.free(arr);

	std.debug.print("Isto não será impresso\n", .{});
}
```

A última linha deste código é profética, ela não será impressa. Isso ocorre porque liberamos a mesma memória duas vezes. Isso é conhecido como double-free e não é válido. Isso pode parecer simples o suficiente para evitar, mas em projetos grandes com ciclos de vida complexos, pode ser difícil rastrear.

A segunda regra é que você não pode liberar a memória da qual você não tem uma referência. Isso pode parecer óbvio, mas nem sempre fica claro quem é responsável por liberá-lo. O seguinte cria uma nova string em minúsculas:

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

fn allocLower(allocator: Allocator, str: []const u8) ![]const u8 {
	var dest = try allocator.alloc(u8, str.len);

	for (str, 0..) |c, i| {
		dest[i] = switch (c) {
			'A'...'Z' => c + 32,
			else => c,
		};
	}

	return dest;
}
```

O código acima é permitido. Mas o seguinte não é:

```zig
// Neste caso, especificamente, nós deveríamos ter usado "std.ascii.eqlIgnoreCase"
fn isSpecial(allocator: Allocator, name: [] const u8) !bool {
	const lower = try allocLower(allocator, name);
	return std.mem.eql(u8, lower, "admin");
}
```

Isso é um vazamento de memória. A memória criada em `allocLower` nunca é liberada. Além disso, uma vez que `isSpecial` retorna, ela nunca pode ser liberada. Em linguagens com coletores de lixo, quando os dados se tornam inacessíveis, eventualmente serão liberados pelo coletor de lixo. Mas no código acima, uma vez que `isSpecial` retorna, perdemos nossa única referência à memória alocada, a variável `lower`. A memória está perdida até que nosso processo seja encerrado. Nossa função pode vazar apenas alguns bytes, mas se for um processo de longa execução e essa função for chamada repetidamente, isso se acumulará e eventualmente ficaremos sem memória.

Pelo menos no caso de dupla liberação, teremos uma falha grave. Vazamentos de memória podem ser insidiosos. Não é apenas que a causa raiz pode ser difícil de identificar. Vazamentos muito pequenos ou vazamentos em código executado raramente podem ser ainda mais difíceis de detectar. Esse é um problema tão comum que o Zig fornece ajuda, como veremos ao falar sobre alocadores.



## Criar _(create)_ & destruir _(destroy)_

O método `alloc` do `std.mem.Allocator` retorna uma fatia com o comprimento que foi passado como o segundo parâmetro. Se você deseja um único valor, usará `create` e `destroy` em vez de `alloc` e `free`. Algumas partes atrás, ao aprender sobre ponteiros, criamos um `User` e tentamos incrementar seu `power`. Aqui está a versão funcional baseada no heap desse código usando `create`:

```zig
const std = @import("std");

pub fn main() !void {
	// novamente, iremos falar sobre alocadores em breve!
	var gpa = std.heap.GeneralPurposeAllocator(.{}){};
	const allocator = gpa.allocator();

	// criar um usuário (User) na memória dinâmica (heap)
	var user = try allocator.create(User);

	// liberar a memória alocada para o usuário ao final deste escopo
	defer allocator.destroy(user);

	user.id = 1;
	user.power = 100;

	// linha adicionada
	levelUp(user);
	std.debug.print("User {d} has power of {d}\n", .{user.id, user.power});
}

fn levelUp(user: *User) void {
	user.power += 1;
}

pub const User = struct {
	id: u64,
	power: i32,
};
```

O método `create` recebe um único parâmetro, o tipo (`T`). Ele retorna um ponteiro para esse tipo ou um erro, ou seja, `!*T`. Talvez você esteja se perguntando o que aconteceria se criássemos nosso usuário (`user`), mas não definíssemos o `id` e/ou `power`. Isso é como definir esses campos como indefinidos e o comportamento é, bom, indefinido.

Quando exploramos ponteiros pendurados, tivemos uma função que retornava incorretamente o endereço do usuário local:

```zig
pub const User = struct {
	fn init(id: u64, power: i32) *User{
		var user = User{
			.id = id,
			.power = power,
		};
		// isto é um ponteiro pendurado/solto (dangling pointer)
		return &user;
	}
};
```

Neste caso, teria feito mais sentido retornar um `User`. Mas, às vezes, você desejará que uma função retorne um ponteiro para algo que ela cria. Você fará isso quando quiser que um tempo de vida seja livre da rigidez da pilha de chamadas. Para resolver nosso ponteiro pendurado acima, poderíamos ter usado o `create`:

```zig
// nosso tipo de retorno mudou, agora que "init" pode falhar nós mudamos: *User -> !*User
fn init(allocator: std.mem.Allocator, id: u64, power: i32) !*User{
	var user = try allocator.create(User);
	user.* = .{
		.id = id,
		.power = power,
	};
	return user;
}
```

Eu introduzi uma nova sintaxe, `user.* = .{...}`. É um pouco estranho, e eu não adoro, mas você verá isso. O lado direito é algo que você já viu: é um inicializador de estrutura com um tipo inferido. Poderíamos ter sido explícitos e usado: `user.* = User{...}`. O lado esquerdo, `user.*`, é como desreferenciamos um ponteiro. `&` pega um `T` e nos dá `*T`. `.*` é o oposto, aplicado a um valor do tipo `*T`, nos dá `T`. Lembre-se de que `create` retorna um `!*User`, então nosso usuário é do tipo `*User`.



## Alocadores

Um dos princípios fundamentais do Zig é a ausência de alocações de memória ocultas _("No hidden memory allocations")_. Dependendo da sua experiência, isso pode não parecer tão especial. Mas é um contraste acentuado com o que você encontrará em C, onde a memória é alocada com a função `malloc` da biblioteca padrão. Em C, se você quiser saber se uma função aloca memória ou não, precisa ler o código-fonte e procurar chamadas para `malloc`.

Zig não possui um alocador padrão. Em todos os exemplos acima, as funções que alocavam memória recebiam um parâmetro `std.mem.Allocator`. Por convenção, este é geralmente o primeiro parâmetro. Toda a biblioteca padrão do Zig, e a maioria das bibliotecas de terceiros, exigem que o chamador forneça um alocador se pretender alocar memória.

Essa explicitação pode se apresentar de duas formas. Em casos simples, o alocador é fornecido em cada chamada de função. Existem muitos exemplos disso, mas `std.fmt.allocPrint` é um que você provavelmente precisará mais cedo ou mais tarde. É semelhante ao `std.debug.print` que usamos, mas aloca e retorna uma string em vez de escrevê-la em stderr:

```zig
const say = std.fmt.allocPrint(allocator, "It's over {d}!!!", .{user.power});
defer allocator.free(say);
```

A outra forma é quando um alocador é passado para `init` e, em seguida, é usado internamente pelo objeto. Vimos isso acima com nossa estrutura `Game`. Isso é menos explícito, já que você forneceu ao objeto um alocador para usar, mas você não sabe quais chamadas de método realmente vão alocar. Essa abordagem é mais prática para objetos de longa duração.

A vantagem de injetar o alocador não é apenas a explicitação, mas também a flexibilidade. `std.mem.Allocator` é uma interface que fornece as funções `alloc`, `free`, `create` e `destroy`, junto com algumas outras. Até agora, só vimos o `std.heap.GeneralPurposeAllocator`, mas outras implementações estão disponíveis na biblioteca padrão ou como bibliotecas de terceiros.

> Zig não tem uma sintaxe elegante para criar interfaces. Um padrão para comportamento de vida de interface são uniões marcadas, embora isso seja relativamente restrito em comparação com interfaces verdadeiras. Outros padrões surgiram e são usados em toda a biblioteca padrão, como no caso de `std.mem.Allocator`. Se você estiver interessado, eu publiquei uma [postagem à parte no blog explicando interfaces](https://www.openmymind.net/Zig-Interfaces/).

Se você estiver construindo uma biblioteca, é melhor aceitar um `std.mem.Allocator` e permitir que os usuários da sua biblioteca decidam qual implementação de alocador usar. Caso contrário, você precisará escolher o alocador correto e, como veremos, essas escolhas não são mutuamente exclusivas. Pode haver boas razões para criar diferentes alocadores dentro do seu programa.



## Alocador de uso geral

Como o nome indica, o `std.heap.GeneralPurposeAllocator` é um alocador versátil "de uso geral", seguro para threads, que pode servir como o alocador principal de sua aplicação. Para muitos programas, este será o único alocador necessário. No início do programa, um alocador é criado e passado para as funções que o precisam. O código de exemplo do meu servidor HTTP é um bom exemplo:

```zig
const std = @import("std");
const httpz = @import("httpz");

pub fn main() !void {
    // criar nosso alocador para uso geral
	var gpa = std.heap.GeneralPurposeAllocator(.{}){};

	// retornar um "std.mem.Allocator" do alocador criado
	const allocator = gpa.allocator();

	// passar nosso alocador para funções e bibliotecas que necessitem dele
	var server = try httpz.Server().init(allocator, .{.port = 5882});

	var router = server.router();
	router.get("/api/user/:id", getUser);

	// bloquear o fluxo de execução (thread) atual
	try server.listen();
}
```

Criamos o `GeneralPurposeAllocator`, obtemos um `std.mem.Allocator` dele e o passamos para a função `init` do servidor HTTP. Em um projeto mais complexo, o alocador seria passado para várias partes do código, cada uma delas possivelmente o passando para suas próprias funções, objetos e dependências.

Você pode notar que a sintaxe em torno da criação de `gpa` é um pouco estranha. O que é isso: `GeneralPurposeAllocator(.{}){}`? São todas coisas que já vimos antes, apenas juntas. `std.heap.GeneralPurposeAllocator` é uma função e, como está usando PascalCase, sabemos que ela retorna um tipo. (Vamos falar mais sobre genéricos na próxima parte). Sabendo que ela retorna um tipo, talvez esta versão mais explícita seja mais fácil de entender:

```zig
const T = std.heap.GeneralPurposeAllocator(.{});
var gpa = T{};

// é o mesmo que:

var gpa = std.heap.GeneralPurposeAllocator(.{}){};
```

Talvez você ainda tenha dúvidas sobre o significado de `.{} `. Isso também é algo que já vimos antes: é um inicializador de estrutura com um tipo implícito. Qual é o tipo e onde estão os campos? O tipo é `std.heap.general_purpose_allocator.Config`, embora ele não seja diretamente exposto assim, que é uma razão pela qual não somos explícitos. Nenhum campo é definido porque a estrutura `Config` define padrões, que estaremos usando. Este é um padrão comum com configurações/opções. Na verdade, vemos isso novamente algumas linhas abaixo quando passamos `. {.port = 5882}` para o `init`. Neste caso, estamos usando o valor padrão para todos os campos, exceto um, a porta.



## std.testing.allocator

Espero que você tenha ficado suficientemente preocupado quando falamos sobre vazamentos de memória e, em seguida, ansioso para aprender mais quando mencionei que o Zig poderia ajudar. Essa ajuda vem do `std.testing.allocator`, que é um `std.mem.Allocator`. Atualmente, ele é implementado usando o `GeneralPurposeAllocator` com integração adicional no test runner do Zig, mas isso é um detalhe de implementação. O importante é que se usarmos `std.testing.allocator` em nossos testes, podemos detectar a maioria dos vazamentos de memória.

Você provavelmente já está familiarizado com arrays dinâmicos, frequentemente chamados de ArrayLists. Em muitas linguagens de programação dinâmicas, todos os arrays são arrays dinâmicos. Os arrays dinâmicos suportam um número variável de elementos. Zig tem um ArrayList genérico adequado, mas criaremos um especificamente para armazenar inteiros e demonstrar a detecção de vazamentos:

```zig
pub const IntList = struct {
	pos: usize,
	items: []i64,
	allocator: Allocator,

	fn init(allocator: Allocator) !IntList {
		return .{
			.pos = 0,
			.allocator = allocator,
			.items = try allocator.alloc(i64, 4),
		};
	}

	fn deinit(self: IntList) void {
		self.allocator.free(self.items);
	}

	fn add(self: *IntList, value: i64) !void {
		const pos = self.pos;
		const len = self.items.len;

		if (pos == len) {
		    // ficamos sem espaço
		    // cria-se um novo "slice" duas vezes maior
			var larger = try self.allocator.alloc(i64, len * 2);

            // copiamos os items que adicionamos previamente para o nosso novo espaço
            @memcpy(larger[0..len], self.items);

			self.items = larger;
		}

		self.items[pos] = value;
		self.pos = pos + 1;
	}
};
```

A parte interessante ocorre em `add` quando `pos == len`, indicando que preenchemos nosso array atual e precisamos criar um maior. Podemos usar `IntList` da seguinte forma:

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

pub fn main() !void {
	var gpa = std.heap.GeneralPurposeAllocator(.{}){};
	const allocator = gpa.allocator();

	var list = try IntList.init(allocator);
	defer list.deinit();

	for (0..10) |i| {
		try list.add(@intCast(i));
	}

	std.debug.print("{any}\n", .{list.items[0..list.pos]});
}
```

O código é executado e imprime o resultado correto. No entanto, mesmo que tenhamos chamado `deinit` em `list`, há um vazamento de memória. Não se preocupe se você não percebeu, porque vamos escrever um teste e usar `std.testing.allocator`:

```zig
const testing = std.testing;
test "IntList: add" {
    // aqui estamos usando "testing.allocator"!
	var list = try IntList.init(testing.allocator);
	defer list.deinit();

	for (0..5) |i| {
		try list.add(@intCast(i+10));
	}

	try testing.expectEqual(@as(usize, 5), list.pos);
	try testing.expectEqual(@as(i64, 10), list.items[0]);
	try testing.expectEqual(@as(i64, 11), list.items[1]);
	try testing.expectEqual(@as(i64, 12), list.items[2]);
	try testing.expectEqual(@as(i64, 13), list.items[3]);
	try testing.expectEqual(@as(i64, 14), list.items[4]);
}
```

> O `@as` é uma função nativa do Zig que realiza coerção de tipo. Se você está se perguntando por que nosso teste teve que usar tantos deles, você não está sozinho. Tecnicamente, é porque o segundo parâmetro, o "atual", é convertido para o primeiro, o "esperado". No código acima, nossos "esperados" são todos do tipo `comptime_int`, o que causa problemas. Muitos, eu incluso, consideram esse [comportamento estranho e infeliz](https://github.com/ziglang/zig/issues/4437).

Se você está acompanhando, coloque o teste no mesmo arquivo que `IntList` e `main`. Os testes em Zig geralmente são escritos no mesmo arquivo, frequentemente próximo ao código que estão testando. Quando usamos `zig test learning.zig` para executar nosso teste, obtemos uma falha incrível:

```
Test [1/1] test.IntList: add... [gpa] (err): memory address 0x101154000 leaked:
/code/zig/learning.zig:26:32: 0x100f707b7 in init (test)
   .items = try allocator.alloc(i64, 2),
                               ^
/code/zig/learning.zig:55:29: 0x100f711df in test.IntList: add (test)
 var list = try IntList.init(testing.allocator);

... MORE STACK INFO ...

[gpa] (err): memory address 0x101184000 leaked:
/code/test/learning.zig:40:41: 0x100f70c73 in add (test)
   var larger = try self.allocator.alloc(i64, len * 2);
                                        ^
/code/test/learning.zig:59:15: 0x100f7130f in test.IntList: add (test)
  try list.add(@intCast(i+10));
```

Temos múltiplos vazamentos de memória. Felizmente, o alocador de teste nos diz exatamente onde a memória com vazamento foi alocada. Você consegue identificar o vazamento agora? Se não, lembre-se de que, em geral, cada `alloc` deve ter um `free` correspondente. Nosso código chama `free` uma vez, em `deinit`. No entanto, `alloc` é chamado uma vez em `init` e, em seguida, toda vez que `add` é chamado e precisamos de mais espaço. Toda vez que alocamos mais espaço, precisamos liberar o `self.items` anterior:

```zig
// código existente
var larger = try self.allocator.alloc(i64, len * 2);
@memcpy(larger[0..len], self.items);

// código adicionado
// liberação das alocações feitas
self.allocator.free(self.items);
```

Adicionando esta última linha, após copiar os elementos para nosso `slice` maior, resolve o problema. Se você executar `zig test learning.zig`, não deverá haver erros.



## ArenaAllocator

O `GeneralPurposeAllocator` é uma escolha padrão razoável porque funciona bem em todos os casos possíveis. No entanto, dentro de um programa, você pode se deparar com padrões de alocação que podem se beneficiar de alocadores mais especializados. Um exemplo é a necessidade de um estado de curta duração que pode ser descartado quando o processamento for concluído. "Parseadores" frequentemente têm tal requisito. Uma função de análise básica pode se parecer com:

```zig
fn parse(allocator: Allocator, input: []const u8) !Something {
	var state = State{
		.buf = try allocator.alloc(u8, 512),
		.nesting = try allocator.alloc(NestType, 10),
	};
	defer allocator.free(state.buf);
	defer allocator.free(state.nesting);

	return parseInternal(allocator, state, input);
}
```

Embora isso não seja muito difícil de gerenciar, `parseInternal` pode precisar de outras alocações de curta duração que precisarão ser liberadas. Como alternativa, poderíamos criar um ArenaAllocator que nos permite liberar todas as alocações de uma só vez:

```zig
fn parse(allocator: Allocator, input: []const u8) !Something {
	// criar um "ArenaAllocator" do alocador providenciado
	var arena = std.heap.ArenaAllocator.init(allocator);

	// isto irá liberar qualquer coisa criada desta arena de memória
	defer arena.deinit();

    // criar um "std.mem.Allocator" da arena, este será o alocador que iremos utilizados internamente
	const aa = arena.allocator();

	var state = State{
		// aqui usamos "aa"!
		.buf = try aa.alloc(u8, 512),

		// aqui usamos "aa"!
		.nesting = try aa.alloc(NestType, 10),
	};

	// estamos passando "aa" aqui, então garantimos que quaisquer outras alocações ocorrerão dentro da arena de memória criada
	return parseInternal(aa, state, input);
}
```

O `ArenaAllocator` recebe um alocador filho, neste caso o alocador que foi passado para `init`, e cria um novo `std.mem.Allocator`. Quando este novo alocador é usado para alocar ou criar memória, não precisamos chamar `free` ou `destroy`. Tudo será liberado quando chamarmos `deinit` na `arena`. Na verdade, o `free` e `destroy` de um `ArenaAllocator` não fazem nada.

O `ArenaAllocator` deve ser usado com cuidado. Como não há maneira de liberar alocações individuais, é necessário ter certeza de que o `deinit` da arena será chamado dentro de um crescimento de memória razoável. Curiosamente, esse conhecimento pode ser interno ou externo. Por exemplo, em nosso esqueleto acima, fazer uso de um ArenaAllocator faz sentido dentro do Parser, já que os detalhes do tempo de vida do estado são uma questão interna.

> Alocadores como o `ArenaAllocator`, que têm um mecanismo para liberar todas as alocações anteriores, podem quebrar a regra de que cada alocação (`alloc`) deve ter uma liberação (`free`) correspondente. No entanto, se você receber um `std.mem.Allocator`, não deve fazer suposições sobre a implementação por de baixo.

O mesmo não pode ser dito para a nossa `IntList`. Ela pode ser usada para armazenar 10 ou 10 milhões de valores. Pode ter uma vida útil medida em milissegundos ou semanas. Ela não está em posição de decidir o tipo de alocador a ser usado. É o código que faz uso de `IntList` que tem esse conhecimento. Originalmente, gerenciamos nossa `IntList` assim:

```zig
var gpa = std.heap.GeneralPurposeAllocator(.{}){};
const allocator = gpa.allocator();

var list = try IntList.init(allocator);
defer list.deinit();
```

Poderíamos ter optado por fornecer um `ArenaAllocator`:

```zig
var gpa = std.heap.GeneralPurposeAllocator(.{}){};
const allocator = gpa.allocator();

var arena = std.heap.ArenaAllocator.init(allocator);
defer arena.deinit();
const aa = arena.allocator();

var list = try IntList.init(aa);

// Honestamente, estou em dúvida se devemos ou não chamar "list.deinit"
// Tecnicamente, não precisamos visto que utilizamos "defer" na chamada acima para "arena.deinit()" arena.deinit() above.
defer list.deinit();

...
```

Não precisamos alterar a `IntList`, pois ela lida apenas com um `std.mem.Allocator`. E se a `IntList` internamente criasse sua própria arena, isso também funcionaria. Não há motivo para você não criar uma arena dentro de uma arena.

Como último exemplo rápido, o servidor HTTP que mencionei acima expõe um alocador de arena na `Response`. Assim que a resposta é enviada, a arena é limpa. A vida útil previsível da arena (do início ao fim da solicitação) a torna uma opção eficiente. Eficiente em termos de desempenho e facilidade de uso.



## FixedBufferAllocator

O último alocador que vamos examinar é o `std.heap.FixedBufferAllocator`, que aloca memória a partir de um buffer (ou seja, `[]u8`) que fornecemos. Este alocador tem dois benefícios principais. Primeiro, uma vez que toda a memória que poderia ser usada é criada antecipadamente, ele é rápido. Segundo, ele limita naturalmente quanto de memória pode ser alocado. Esse limite rígido também pode ser visto como uma desvantagem. Outra desvantagem é que `free` e `destroy` só funcionarão no último item alocado/criado (pense em uma pilha). Chamar `free` em uma alocação que não é a última é seguro, mas não fará nada.

```zig
const std = @import("std");

pub fn main() !void {
	var buf: [150]u8 = undefined;
	var fa = std.heap.FixedBufferAllocator.init(&buf);
	defer fa.reset();

	const allocator = fa.allocator();

	const json = try std.json.stringifyAlloc(allocator, .{
		.this_is = "an anonymous struct",
		.above = true,
		.last_param = "are options",
	}, .{.whitespace = .indent_2});

	std.debug.print("{s}\n", .{json});
}
```

O código acima imprime:

```
{
  "this_is": "an anonymous struct",
  "above": true,
  "last_param": "are options"
}
```

Mas mude nosso `buf` para ser um `[120]u8` e você obterá um erro de `OutOfMemory`.

Um padrão comum com `FixedBufferAllocators` e, em menor grau, com `ArenaAllocators`, é usar `reset` neles e reutilizá-los. Isso libera todas as alocações anteriores e permite que o alocador seja reutilizado.



---

Ao não ter um alocador padrão, Zig é tanto transparente quanto flexível em relação às alocações. A interface `std.mem.Allocator` é poderosa, permitindo que alocadores especializados envolvam os mais gerais, como vimos com o `ArenaAllocator`.

De maneira mais geral, o poder e as responsabilidades associadas às alocações de heap são esperançosamente evidentes. A capacidade de alocar memória de tamanho arbitrário com uma vida útil arbitrária é essencial para a maioria dos programas.

No entanto, devido à complexidade que vem com a memória dinâmica, é bom ficar de olho em alternativas. Por exemplo, acima usamos `std.fmt.allocPrint`, mas a biblioteca padrão também possui um `std.fmt.bufPrint`. Este último recebe um _buffer_ em vez de um alocador:

```zig
const std = @import("std");

pub fn main() !void {
	const name = "Leto";

	var buf: [100]u8 = undefined;
	const greeting = try std.fmt.bufPrint(&buf, "Hello {s}", .{name});

	std.debug.print("{s}\n", .{greeting});
}
```

Essa API transfere a responsabilidade do gerenciamento de memória para o chamador. Se tivéssemos um `name` mais longo ou um `buf` menor, nosso `bufPrint` poderia retornar um erro `NoSpaceLeft`. Mas existem muitos cenários em que um aplicativo possui limites conhecidos, como um comprimento máximo de nome. Nesses casos, `bufPrint` é mais seguro e rápido.

Outra possível alternativa para alocações dinâmicas é transmitir dados para um `std.io.Writer`. Assim como nosso `Allocator`, `Writer` é uma interface implementada por muitos tipos, como arquivos. Acima, usamos `stringifyAlloc` para serializar JSON em uma string alocada dinamicamente. Poderíamos ter usado `stringify` e fornecido um `Writer`:

```zig
pub fn main() !void {
	const out = std.io.getStdOut();

	try std.json.stringify(.{
		.this_is = "an anonymous struct",
		.above = true,
		.last_param = "are options",
	}, .{.whitespace = .indent_2}, out.writer());
}
```

> Enquanto alocadores são frequentemente fornecidos como o primeiro parâmetro de uma função, os `writers` geralmente são os últimos. ಠ_ಠ

Em muitos casos, envolver nosso _writer_ em um `std.io.BufferedWriter` proporcionaria um bom impulso de desempenho.

O objetivo não é eliminar todas as alocações dinâmicas. Isso não funcionaria, já que essas alternativas só fazem sentido em casos específicos. Mas agora você tem muitas opções à sua disposição. Desde frames de pilha até um alocador de propósito geral, e todas as coisas intermediárias, como buffers estáticos, writers de streaming e alocadores especializados.
