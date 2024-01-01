# Programando em Zig

Com grande parte da linguagem agora abordada, vamos concluir revisando alguns tópicos e explorando alguns aspectos mais práticos do uso do Zig. Ao fazer isso, vamos introduzir mais da biblioteca padrão e apresentar trechos de código menos triviais.



## Ponteiros pendurados/soltos _(dangling pointers)_

Vamos começar examinando mais exemplos de ponteiros pendurados. Isso pode parecer algo estranho para focar, mas se você estiver vindo de uma linguagem com coleta de lixo, isso provavelmente será o maior desafio que você enfrentará.

Você consegue descobrir qual será a saída do seguinte código?

```zig
const std = @import("std");

pub fn main() !void {
	var gpa = std.heap.GeneralPurposeAllocator(.{}){};
	const allocator = gpa.allocator();

	var lookup = std.StringHashMap(User).init(allocator);
	defer lookup.deinit();

	const goku = User{.power = 9001};

	try lookup.put("Goku", goku);

    // retorna um opcional, .? iria levantar um erro caso "Goku"
    // não estivesse em nosso dicionário (hashmap)
	const entry = lookup.getPtr("Goku").?;

	std.debug.print("Goku's power is: {d}\n", .{entry.power});

    // retorna verdadeiro/falso dependendo se o item foi removido ou não
	_ = lookup.remove("Goku");

	std.debug.print("Goku's power is: {d}\n", .{entry.power});
}

const User = struct {
	power: i32,
};
```

Ao executar isto, recebi como resultado:

```
Goku's power is: 9001
Goku's power is: -1431655766
```

Este código introduz o `std.StringHashMap` genérico do Zig, que é uma versão especializada do `std.AutoHashMap` com o tipo de chave definido como `[]const u8`. Mesmo que você não tenha certeza do que está acontecendo, é uma boa suposição que a minha saída está relacionada ao fato de que nosso segundo `print` ocorre após o `remove` da entrada de `lookup`. Comente a chamada para `remove`, e a saída será normal.

A chave para entender o código acima é estar ciente de onde os dados/memória existem, ou, em outras palavras, quem é o proprietário. Lembre-se de que os argumentos do Zig são passados por valor, ou seja, passamos uma cópia [superficial] do valor. O `User` em nosso `lookup` não é a mesma memória referenciada por `goku`. Nosso código acima tem dois usuários, cada um com seu próprio proprietário. `goku` é de propriedade do `main`, e sua cópia é de propriedade de `lookup`.

O método `getPtr` retorna um ponteiro para o valor no mapa, em nosso caso, ele retorna um `*User`. Aqui está o problema, `remove` torna nosso ponteiro de `entry` inválido. Neste exemplo, a proximidade de `getPtr` e `remove` torna o problema um pouco óbvio. Mas não é difícil imaginar código chamando `remove` sem saber que uma referência à entrada está sendo mantida em outro lugar.

> Quando escrevi este exemplo, não tinha certeza do que aconteceria. Era possível que `remove` fosse implementado definindo uma marcação interna, adiando a remoção real até um evento posterior. Se esse fosse o caso, o código acima poderia ter "funcionado" em nossos casos simples, mas teria falhado com um uso mais complicado. Isso soa aterrorizantemente difícil de depurar.

Além de não chamar `remove`, podemos corrigir isso de algumas maneiras diferentes. A primeira é que poderíamos usar `get` em vez de `getPtr`. Isso retornaria um `User` em vez de um `*User` e, portanto, retornaria uma cópia do valor em `lookup`. Teríamos então três `Users`.

1. O original `goku`, vinculado à função.
2. A cópia em `lookup`, de propriedade do `lookup`.
3. E uma cópia de nossa cópia, `entry`, também vinculada à função.

Como `entry` seria agora sua própria cópia independente do usuário, removê-lo de `lookup` não o invalidaria.

Outra opção é alterar o tipo de `lookup` de `StringHashMap(User)` para `StringHashMap(*const User)`. Este código funciona:

```zig
const std = @import("std");

pub fn main() !void {
	var gpa = std.heap.GeneralPurposeAllocator(.{}){};
	const allocator = gpa.allocator();

	// User -> *const User
	var lookup = std.StringHashMap(*const User).init(allocator);
	defer lookup.deinit();

	const goku = User{.power = 9001};

	// goku -> &goku
	try lookup.put("Goku", &goku);

	// getPtr -> get
	const entry = lookup.get("Goku").?;

	std.debug.print("Goku's power is: {d}\n", .{entry.power});
	_ = lookup.remove("Goku");
	std.debug.print("Goku's power is: {d}\n", .{entry.power});
}

const User = struct {
	power: i32,
};
```

Existem várias sutilezas no código acima. Em primeiro lugar, agora temos um único `User`, o `goku`. O valor em `lookup` e `entry` são ambas referências ao `goku`. Nossa chamada para `remove` ainda remove o valor de nosso `lookup`, mas esse valor é apenas o endereço de `user`, não é o `user` em si. Se tivéssemos mantido `getPtr`, obteríamos um `**User` inválido, inválido por causa do `remove`. Em ambas as soluções, tivemos que usar `get` em vez de `getPtr`, mas neste caso, estamos apenas copiando o endereço, não o `User` completo. Para objetos grandes, isso pode fazer uma diferença significativa.

Com tudo em uma única função e um valor pequeno como `User`, isso ainda parece um problema criado artificialmente. Precisamos de um exemplo que torne a propriedade dos dados uma preocupação imediata.



## Propriedade _(ownership)_

Eu adoro mapas de hash porque são algo que todos conhecem e usam. Eles também têm muitos casos de uso diferentes, a maioria dos quais você provavelmente experimentou em primeira mão. Embora possam ser usados para pesquisas de curta duração, muitas vezes são de longa duração e, portanto, exigem valores igualmente duradouros.

Este código preenche nosso mapa (`lookup`) com nomes que você insere no terminal. Um nome vazio interrompe o loop do prompt. Finalmente, ele detecta se "Leto" foi um dos nomes fornecidos.

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() !void {
	var gpa = std.heap.GeneralPurposeAllocator(.{}){};
	const allocator = gpa.allocator();

	var lookup = std.StringHashMap(User).init(allocator);
	defer lookup.deinit();

	// stdin é um std.io.Reader
	// o oposto de um std.io.Writer, que já vimos
	const stdin = std.io.getStdIn().reader();

	// stdout é um std.io.Writer
	const stdout = std.io.getStdOut().writer();

	var i: i32 = 0;
	while (true) : (i += 1) {
		var buf: [30]u8 = undefined;
		try stdout.print("Please enter a name: ", .{});
		if (try stdin.readUntilDelimiterOrEof(&buf, '\n')) |line| {
			var name = line;
			if (builtin.os.tag == .windows) {
				// No Windows as linhas são terminadas com \r\n.
				// Então temos que remover o \r
				name = std.mem.trimRight(u8, name, "\r");
			}
			if (name.len == 0) {
				break;
			}
			try lookup.put(name, .{.power = i});
		}
	}

	const has_leto = lookup.contains("Leto");
	std.debug.print("{any}\n", .{has_leto});
}

const User = struct {
	power: i32,
};
```

O código é sensível a maiúsculas e minúsculas, mas não importa o quão perfeitamente digitamos "Leto", `contains` sempre retorna `false`. Vamos depurar isso iterando através de `lookup` e exibindo as chaves e valores:

```zig
// Acrescente este código após a iteração do "while"

var it = lookup.iterator();
while (it.next()) |kv| {
	std.debug.print("{s} == {any}\n", .{kv.key_ptr.*, kv.value_ptr.*});
}
```

Esse padrão de iterador é comum em Zig e depende da sinergia entre `while` e tipos opcionais. Nosso item do iterador retorna ponteiros para a chave e o valor, por isso os desreferenciamos com `.*` para acessar o valor real em vez do endereço. A saída dependerá do que você inserir, mas eu obtive:

```
Please enter a name: Paul
Please enter a name: Teg
Please enter a name: Leto
Please enter a name:

�� == learning.User{ .power = 1 }

��� == learning.User{ .power = 0 }

��� == learning.User{ .power = 2 }
false
```

Os valores parecem estar corretos, mas não as chaves. Se você não tem certeza do que está acontecendo, provavelmente é minha culpa. Antes, intencionalmente, desviei sua atenção. Eu disse que os mapas de hash são frequentemente de longa duração e, portanto, exigem valores de longa duração. A verdade é que eles exigem valores e chaves de longa duração! Observe que `buf` é definido dentro do nosso loop `while`. Quando chamamos o `put`, estamos dando ao nosso mapa de hash uma chave que tem uma vida útil muito mais curta do que o próprio mapa de hash. Mover `buf` para fora do loop `while` resolve nosso problema de tempo de vida, mas esse buffer é reutilizado em cada iteração. Ainda não funcionará porque estamos mutando os dados da chave subjacente.

Para o código acima, há realmente apenas uma solução: nosso `lookup` deve assumir a propriedade das chaves. Precisamos adicionar uma linha e alterar outra:

```zig
// substitua o "lookup.put" existente por estas duas linhas
const owned_name = try allocator.dupe(u8, name);

// name -> owned_name
try lookup.put(owned_name, .{.power = i});
```

`dupe` é um método de `std.mem.Allocator` que ainda não vimos antes. Ele aloca uma duplicata do valor fornecido. O código agora funciona porque nossas chaves, agora no heap, têm uma vida útil mais longa que `lookup`. Na verdade, fizemos um trabalho bom demais em estender a vida dessas strings: introduzimos vazamentos de memória.

Você pode ter pensado que, quando chamamos `lookup.deinit`, nossas chaves e valores seriam liberados para nós. Mas não há uma solução única que `StringHashMap` poderia usar. Primeiro, as chaves podem ser literais de string, que não podem ser liberadas. Segundo, elas podem ter sido criadas com um alocador diferente. Finalmente, embora mais avançado, há casos legítimos em que as chaves podem não ser de propriedade do mapa de hash.

A única solução é liberar as chaves por nós mesmos. Neste ponto, provavelmente faria sentido criar nosso próprio tipo `UserLookup` e encapsular essa lógica de limpeza em nossa função `deinit`. Vamos manter as coisas bagunçadas:

```zig
// substitua:
//   defer lookup.deinit();
// por:
defer {
	var it = lookup.keyIterator();
	while (it.next()) |key| {
		allocator.free(key.*);
	}
	lookup.deinit();
}
```

Nossa lógica de `defer`, a primeira que vimos com um bloco, libera cada chave e, em seguida, desinicializa `lookup`. Estamos usando `keyIterator` para iterar apenas sobre as chaves. O valor do iterador é um ponteiro para a entrada da chave no mapa de hash, um `*[]const u8`. Queremos liberar o valor real, já que é isso que alocamos via `dupe`, então desreferenciamos o valor usando `.*`.

Eu prometo, terminamos de falar sobre ponteiros pendentes e gerenciamento de memória. O que discutimos ainda pode estar pouco claro ou muito abstrato. Está tudo bem revisitar isso quando você tiver um problema mais prático para resolver. Dito isso, se você planeja escrever algo não trivial, é quase certo que precisará dominar esse tópico. Quando se sentir preparado, sugiro que pegue o exemplo do loop de prompt e brinque com ele por conta própria. Introduza um tipo `UserLookup` que encapsule toda a gestão de memória que tivemos que fazer. Tente ter valores `*User` em vez de `User`, criando os usuários no heap e liberando-os como fizemos com as chaves. Escreva testes que cubram sua nova estrutura, usando o `std.testing.allocator` para garantir que não está vazando memória.



## ArrayList

Você ficará feliz em saber que pode esquecer sobre nosso `IntList` e a alternativa genérica que criamos. Zig possui uma implementação adequada de array dinâmico: `std.ArrayList(T)`.

É algo bastante padrão, mas é uma estrutura de dados tão comumente necessária e utilizada que vale a pena vê-la em ação:

```zig
const std = @import("std");
const builtin = @import("builtin");
const Allocator = std.mem.Allocator;

pub fn main() !void {
	var gpa = std.heap.GeneralPurposeAllocator(.{}){};
	const allocator = gpa.allocator();

	var arr = std.ArrayList(User).init(allocator);
	defer {
		for (arr.items) |user| {
			user.deinit(allocator);
		}
		arr.deinit();
	}

	// stdin é um std.io.Reader
	// o oposto de um std.io.Writer, que já vimos
	const stdin = std.io.getStdIn().reader();

	// stdout é um std.io.Writer
	const stdout = std.io.getStdOut().writer();

	var i: i32 = 0;
	while (true) : (i += 1) {
		var buf: [30]u8 = undefined;
		try stdout.print("Please enter a name: ", .{});
		if (try stdin.readUntilDelimiterOrEof(&buf, '\n')) |line| {
			var name = line;
			if (builtin.os.tag == .windows) {
				// No Windows as linhas são terminadas com \r\n.
				// Temos que remover o \r
				name = std.mem.trimRight(u8, name, "\r");
			}
			if (name.len == 0) {
				break;
			}
			const owned_name = try allocator.dupe(u8, name);
			try arr.append(.{.name = owned_name, .power = i});
		}
	}

	var has_leto = false;
	for (arr.items) |user| {
		if (std.mem.eql(u8, "Leto", user.name)) {
			has_leto = true;
			break;
		}
	}

	std.debug.print("{any}\n", .{has_leto});
}

const User = struct {
	name: []const u8,
	power: i32,

	fn deinit(self: User, allocator: Allocator) void {
		allocator.free(self.name);
	}
};
```

Acima está uma reprodução do nosso código de hash map, mas usando um `ArrayList(User)`. Todas as mesmas regras de tempo de vida e gerenciamento de memória se aplicam. Observe que ainda estamos criando um `dupe` do nome e ainda estamos liberando cada nome antes de desinicializar (`deinit`) o `ArrayList`.

Este é um bom momento para destacar que Zig não possui propriedades ou campos privados. Você pode ver isso quando acessamos `arr.items` para iterar pelos valores. A razão para não ter propriedades é eliminar uma fonte de surpresas. Em Zig, se parece com um acesso a campo, é um acesso a campo. Pessoalmente, acho que a falta de campos privados é um erro, mas certamente é algo com o qual podemos lidar. Tenho usado o prefixo de sublinhado nos campos para sinalizar "uso interno apenas".

Porque a string "type" é uma `[]u8` ou `[]const u8`, um `ArrayList(u8)` é o tipo apropriado para um construtor de strings, como o `StringBuilder` do .NET ou o `strings.Builder` do Go. Na verdade, você frequentemente usará isso quando uma função receber um `Writer` e você quiser uma string. Anteriormente, vimos um exemplo que usava `std.json.stringify` para imprimir JSON no stdout. Aqui está como você usaria um `ArrayList(u8)` para armazená-lo em uma variável:

```zig
const std = @import("std");

pub fn main() !void {
	var gpa = std.heap.GeneralPurposeAllocator(.{}){};
	const allocator = gpa.allocator();

	var out = std.ArrayList(u8).init(allocator);
	defer out.deinit();

	try std.json.stringify(.{
		.this_is = "an anonymous struct",
		.above = true,
		.last_param = "are options",
	}, .{.whitespace = .indent_2}, out.writer());

	std.debug.print("{s}\n", .{out.items});
}
```



## Anytype

Na parte 1, falamos brevemente sobre `anytype`. É uma forma bastante útil de duck-typing em tempo de compilação. Aqui está um logger simples:

```zig
pub const Logger = struct {
	level: Level,

	// "error" é reservado, nomes dentro de @"..." sempre serão
	// tratados como identificadores
	const Level = enum {
		debug,
		info,
		@"error",
		fatal,
	};

	fn info(logger: Logger, msg: []const u8, out: anytype) !void {
		if (@intFromEnum(logger.level) <= @intFromEnum(Level.info)) {
			try out.writeAll(msg);
		}
	}
};
```

O parâmetro de saída (`out`) de nossa função `info` tem o tipo `anytype`. Isso significa que nosso `Logger` pode registrar mensagens em qualquer estrutura que tenha um método `writeAll` aceitando um `[]const u8` e retornando um `!void`. Isso não é uma característica em tempo de execução. A verificação de tipo ocorre em tempo de compilação e, para cada tipo usado, uma função corretamente tipada é criada. Se tentarmos chamar `info` com um tipo que não tenha todas as funções necessárias (neste caso, apenas `writeAll`), receberemos um erro de compilação:

```zig
var l = Logger{.level = .info};
try l.info("sever started", true);
```

Dando-nos: nenhum campo ou função de membro chamado "writeAll" em "bool" _(no field or member function named 'writeAll' in 'bool')_. Usar o `writer` de um `ArrayList(u8)` funciona:

```zig
pub fn main() !void {
	var gpa = std.heap.GeneralPurposeAllocator(.{}){};
	const allocator = gpa.allocator();

	var l = Logger{.level = .info};

	var arr = std.ArrayList(u8).init(allocator);
	defer arr.deinit();

	try l.info("sever started", arr.writer());
	std.debug.print("{s}\n", .{arr.items});
}
```

Uma grande desvantagem do `anytype` é a documentação. Aqui está a assinatura da função `std.json.stringify` que usamos algumas vezes:

```zig
// Eu ODEIO definições de função em várias linhas
// Mas farei uma exceção para o guia
// visto que você pode estar lendo em uma tela pequena.

fn stringify(
	value: anytype,
	options: StringifyOptions,
	out_stream: anytype
) @TypeOf(out_stream).Error!void
```

O primeiro parâmetro, `value: anytype`, é meio óbvio. É o valor a ser serializado e pode ser qualquer coisa (na verdade, há algumas coisas que o serializador JSON do Zig não pode serializar). Podemos supor que `out_stream` é onde escrever o JSON, mas sua suposição é tão boa quanto a minha sobre quais métodos ela precisa implementar. A única maneira de descobrir é ler o código-fonte ou, alternativamente, passar um valor fictício e usar os erros do compilador como nossa documentação. Isso é algo que pode ser aprimorado com geradores automáticos de documentação melhores. Mas, não pela primeira vez, eu gostaria que o Zig tivesse interfaces.



## @TypeOf

Nas partes anteriores, usamos `@TypeOf` para nos ajudar a examinar o tipo de várias variáveis. A partir do nosso uso, você poderia pensar que ele retorna o nome do tipo como uma string. No entanto, dado que é uma função PascalCase, você deve saber melhor: ela retorna um tipo (`type`).

Uma das minhas utilizações favoritas de `anytype` é combiná-la com as funções internas `@TypeOf` e `@hasField` para escrever ajudantes de teste. Embora todo tipo `User` que vimos tenha sido muito simples, peço que você imagine uma estrutura mais complexa com muitos campos. Em muitos de nossos testes, precisamos de um `User`, mas queremos especificar apenas os campos relevantes para o teste. Vamos criar uma `userFactory`:

```zig
fn userFactory(data: anytype) User {
	const T = @TypeOf(data);
	return .{
		.id = if (@hasField(T, "id")) data.id else 0,
		.power = if (@hasField(T, "power")) data.power else 0,
		.active  = if (@hasField(T, "active")) data.active else true,
		.name  = if (@hasField(T, "name")) data.name else "",
	};
}

pub const User = struct {
	id: u64,
	power: u64,
	active: bool,
	name: [] const u8,
};
```

Um usuário padrão pode ser criado chamando `userFactory(.{})`, ou podemos substituir campos específicos com `userFactory(.{.id = 100, .active = false})`. É um pequeno padrão, mas eu realmente gosto dele. Também é um bom passo inicial para o mundo da metaprogramação.

Mais comumente, `@TypeOf` é combinado com `@typeInfo`, que retorna um `std.builtin.Type`. Este é um poderoso sindicato rotulado que descreve totalmente um tipo. A função `std.json.stringify` usa recursivamente isso no valor (`value`) fornecido para descobrir como serializá-lo.



## Zig Build

Se você leu este guia inteiro esperando obter insights sobre a configuração de projetos mais complexos, com várias dependências e vários destinos, você está prestes a ficar desapontado. O Zig possui um sistema de construção poderoso, tanto que um número crescente de projetos não relacionados ao Zig está fazendo uso dele, como o libsodium. Infelizmente, todo esse poder significa que, para necessidades mais simples, ele não é o mais fácil de usar ou entender.

> A verdade é que eu não entendo bem o sistema de construção do Zig o suficiente para explicá-lo.

Ainda assim, podemos pelo menos ter uma visão geral breve. Para executar nosso código Zig, usamos `zig run learning.zig`. Uma vez, também usamos `zig test learning.zig` para executar um teste. Os comandos `run` e `test` são bons para brincar, mas é o comando `build` que você precisará para algo mais complexo. O comando `build` depende de um arquivo `build.zig` com o ponto de entrada `build` especial. Aqui está um esqueleto:

```zig
// build.zig

const std = @import("std");

pub fn build(b: *std.Build) !void {
	_ = b;
}
```

Cada compilação possui uma etapa padrão de "instalação", que você pode agora executar com `zig build install`, mas como nosso arquivo está principalmente vazio, você não obterá artefatos significativos. Precisamos informar à nossa compilação sobre o ponto de entrada do nosso programa, que está em `learning.zig`:

```zig
const std = @import("std");

pub fn build(b: *std.Build) !void {
	const target = b.standardTargetOptions(.{});
	const optimize = b.standardOptimizeOption(.{});

	// executável de configuração
	const exe = b.addExecutable(.{
		.name = "learning",
		.target = target,
		.optimize = optimize,
		.root_source_file = .{ .path = "learning.zig" },
	});
	b.installArtifact(exe);
}
```

Agora, se você executar `zig build install`, obterá um binário em `./zig-out/bin/learning`. Usar os destinos e otimizações padrão nos permite substituir o padrão usando argumentos de linha de comando. Por exemplo, para construir uma versão otimizada para o tamanho do nosso programa para Windows, faríamos:

```
zig build install -Doptimize=ReleaseSmall -Dtarget=x86_64-windows-gnu
```

Um executável geralmente terá dois passos adicionais, além do "install" padrão: "run" e "test". Uma biblioteca pode ter um único passo "test". Para uma execução básica sem argumentos, precisamos adicionar quatro linhas ao final de nosso arquivo build.zig:

```zig
// adicionar depois: b.installArtifact(exe);

const run_cmd = b.addRunArtifact(exe);
run_cmd.step.dependOn(b.getInstallStep());

const run_step = b.step("run", "Start learning!");
run_step.dependOn(&run_cmd.step);
```

Isso cria duas dependências por meio das duas chamadas para `dependOn`. A primeira associa nosso novo comando "run" ao passo de instalação incorporado. A segunda associa o passo "run" ao nosso recém-criado comando "run". Você pode estar se perguntando por que precisa de um comando `run` além de um passo `run`. Eu acredito que essa separação existe para dar suporte a configurações mais complicadas: passos que dependem de vários comandos ou comandos que são usados em vários passos. Se você executar `zig build --help` e rolar para o topo, verá nosso novo passo "run". Agora você pode executar o programa executando `zig build run`.

Para adicionar um passo "test", você duplicará a maior parte do código `run` que acabamos de adicionar, mas em vez de `b.addExecutable`, você iniciará as coisas com `b.addTest`:

```zig
const tests = b.addTest(.{
	.target = target,
	.optimize = optimize,
	.root_source_file = .{ .path = "learning.zig" },
});

const test_cmd = b.addRunArtifact(tests);
test_cmd.step.dependOn(b.getInstallStep());
const test_step = b.step("test", "Run the tests");
test_step.dependOn(&test_cmd.step);
```

Nomeamos esse passo como "test". Executar `zig build --help` agora deve mostrar outro passo disponível, "test". Como não temos nenhum teste, é difícil dizer se isso está funcionando ou não. No arquivo `learning.zig`, adicione o seguinte:

```zig
test "dummy build test" {
	try std.testing.expectEqual(false, true);
}
```

Agora, ao executar `zig build test`, você deve obter uma falha no teste. Se corrigir o teste e executar `zig build test` novamente, você não obterá nenhuma saída. Por padrão, o executor de testes do Zig só produz saída em caso de falha. Use `zig build test --summary all` se, assim como eu, você sempre quiser um resumo, seja para indicar sucesso ou falha.

Essa é a configuração mínima necessária para começar. Mas fique tranquilo sabendo que, se precisar construir algo, Zig provavelmente pode lidar com isso. Por fim, você pode, e provavelmente deve, usar `zig init-exe` ou `zig init-lib` dentro do diretório do seu projeto para que o Zig crie um arquivo `build.zig` bem documentado para você.



## Dependências de bibliotecas de terceiros

O sistema de gerenciamento de pacotes embutido no Zig é relativamente novo e, como consequência, possui algumas arestas ásperas. Embora haja espaço para melhorias, ele é utilizável como está. Existem duas partes que precisamos examinar: criar um pacote e usar pacotes. Vamos passar por isso em detalhes.

Primeiro, crie uma nova pasta chamada `calc` e crie três arquivos. O primeiro é `add.zig`, com o seguinte conteúdo:

```zig
// Ah, uma lição oculta, veja o tipo de b
// e o tipo de retorno!!

pub fn add(a: anytype, b: @TypeOf(a)) @TypeOf(a) {
	return a + b;
}

const testing = @import("std").testing;
test "add" {
	try testing.expectEqual(@as(i32, 32), add(30, 2));
}
```

É um pouco bobo, um pacote inteiro apenas para somar dois valores, mas isso nos permitirá focar no aspecto do empacotamento. Em seguida, adicionaremos outro igualmente bobo: `calc.zig`:

```zig
pub const add = @import("add.zig").add;

test {
    // Por padrão, apenas testes em um determinado arquivo
    // são incluídos. Esta linha de código mágica irá
    // causar uma referência para todos os contêineres
    // que serão testados.
	@import("std").testing.refAllDecls(@This());
}
```

Estamos dividindo isso entre `calc.zig` e `add.zig` para mostrar que o comando `zig build` automaticamente compilará e empacotará todos os arquivos do nosso projeto. Finalmente, podemos adicionar um `build.zig`:

```zig
const std = @import("std");

pub fn build(b: *std.Build) !void {
	const target = b.standardTargetOptions(.{});
	const optimize = b.standardOptimizeOption(.{});

	const tests = b.addTest(.{
		.target = target,
		.optimize = optimize,
		.root_source_file = .{ .path = "calc.zig" },
	});

	const test_cmd = b.addRunArtifact(tests);
	test_cmd.step.dependOn(b.getInstallStep());
	const test_step = b.step("test", "Run the tests");
	test_step.dependOn(&test_cmd.step);
}
```

Isso é tudo uma repetição do que vimos na seção anterior. Com isso, você pode executar `zig build test --summary all`.

De volta ao nosso projeto de aprendizado (`learning`) e ao nosso arquivo `build.zig` anteriormente criado. Vamos começar adicionando nosso `calc` local como uma dependência. Precisamos fazer três adições. Primeiro, vamos criar um módulo apontando para o nosso `calc.zig`:

```zig
// Você pode colocar isto próximo ao tipo da
// função, antes da chamada para "addExecutable".

const calc_module = b.addModule("calc", .{
	.source_file = .{ .path = "CAMINHO_PARA_O_PROJETO_CALC/calc.zig" },
});
```

Você precisará ajustar o caminho para `calc.zig`. Agora, precisamos adicionar este módulo tanto para a variável `exe` quanto para `tests`:

```zig
const exe = b.addExecutable(.{
	.name = "learning",
	.target = target,
	.optimize = optimize,
	.root_source_file = .{ .path = "learning.zig" },
});
// adicione isto
exe.addModule("calc", calc_module);
b.installArtifact(exe);

....

const tests = b.addTest(.{
	.target = target,
	.optimize = optimize,
	.root_source_file = .{ .path = "learning.zig" },
});
// adicione isto
tests.addModule("calc", calc_module);
```

De dentro do projeto, agora você é capaz de usar `@import("calc")`:

```zig
const calc = @import("calc");
...
calc.add(1, 2);
```

Adicionar uma dependência remota exige um pouco mais de esforço. Primeiro, precisamos voltar ao projeto `calc` e definir um módulo. Você pode pensar que o projeto em si é um módulo, mas um projeto pode expor vários módulos, então precisamos criá-lo explicitamente. Usamos o mesmo `addModule`, mas descartamos o valor de retorno. Simplesmente chamar o `addModule` é suficiente para definir o módulo que outros projetos poderão importar.

```zig
_ = b.addModule("calc", .{
	.source_file = .{ .path = "calc.zig" },
});
```

Esta é a única alteração que precisamos fazer em nossa biblioteca. Por ser um exercício de ter uma dependência remota, eu enviei este projeto `calc` para o GitHub para que possamos importá-lo para o nosso projeto de aprendizado. Ele está disponível em https://github.com/karlseguin/calc.zig.

De volta ao nosso projeto de aprendizado, precisamos de um novo arquivo, `build.zig.zon`. "ZON" significa _Zig Object Notation_ e permite que dados Zig sejam expressos em um formato legível por humanos e que esse formato legível por humanos seja transformado em código Zig. O conteúdo do `build.zig.zon` será:

```
.{
  .name = "learning",
  .paths = .{""},
  .version = "0.0.0",
  .dependencies = .{
    .calc = .{
      .url = "https://github.com/karlseguin/calc.zig/archive/e43c576da88474f6fc6d971876ea27effe5f7572.tar.gz",
      .hash = "12ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff"
    },
  },
}
```

Há dois valores questionáveis neste arquivo, o primeiro é `e43c576da88474f6fc6d971876ea27effe5f7572` dentro da `url`. Este é simplesmente o hash do commit do git. O segundo é o valor de `hash`. Até onde eu sei, atualmente não há uma ótima maneira de determinar qual deve ser esse valor, então usamos um valor fictício por enquanto.

Para usar essa dependência, precisamos fazer uma alteração em nosso `build.zig`:

```zig
// substitua isto:
const calc_module = b.addModule("calc", .{
	.source_file = .{ .path = "calc/calc.zig" },
});

// por isso:
const calc_dep = b.dependency("calc", .{.target = target,.optimize = optimize});
const calc_module = calc_dep.module("calc");
```

Em `build.zig.zon`, nomeamos a dependência como `calc`, e é essa dependência que estamos carregando aqui. De dentro dessa dependência, estamos pegando o módulo chamado `calc`, que é o que nomeamos no `build.zig` do `calc`.

Se você tentar executar `zig build test`, deve ver um erro:

```
error: hash mismatch:
expected:
12ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff,

found:
122053da05e0c9348d91218ef015c8307749ef39f8e90c208a186e5f444e818672d4
```

Copie e cole o hash correto de volta no `build.zig.zon` e tente executar `zig build test` novamente. Agora, tudo deve estar funcionando.

Pode parecer um pouco complicado, e espero que as coisas se tornem mais simples. Mas, na maior parte, é algo que você pode copiar e colar de outros projetos e, uma vez configurado, você pode prosseguir.

Um aviso, descobri que o cache de dependências do Zig é bastante agressivo. Se você tentar atualizar uma dependência, mas Zig não parece detectar a mudança... bom, eu costumo apagar a pasta `zig-cache` do projeto, bem como `~/.cache/zig`.



---

Cobrimos muitos tópicos, explorando algumas estruturas de dados fundamentais e reunindo grandes partes das seções anteriores. Nosso código ficou um pouco mais complexo, concentrando-se menos na sintaxe específica e parecendo mais com código real. Estou animado com a possibilidade de que, apesar dessa complexidade, o código tenha feito sentido na maior parte. Se não fez, não desista. Escolha um exemplo, introduza erros, adicione instruções de impressão, escreva alguns testes para ele. Mexa diretamente com o código, criando o seu, e depois volte para ler as partes que não fizeram sentido inicialmente.
