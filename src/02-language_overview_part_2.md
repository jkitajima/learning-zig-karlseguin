# Visão Geral da Linguagem - Parte 2

Esta parte continua de onde a anterior parou: familiarizando-nos com a linguagem. Vamos explorar o fluxo de controle e tipos do Zig além das estruturas. Juntamente com a primeira parte, teremos coberto a maior parte da sintaxe da linguagem, permitindo-nos abordar mais aspectos da linguagem e da biblioteca padrão.



## Fluxo de controle

O fluxo de controle em Zig é provavelmente familiar, mas com sinergias adicionais em relação a aspectos da linguagem que ainda não exploramos. Vamos começar com uma visão geral rápida do fluxo de controle e voltaremos a isso quando discutirmos recursos que geram comportamentos especiais de fluxo de controle.

Você notará que, em vez dos operadores lógicos `&&` e `||`, usamos `and` e `or`. Como na maioria das linguagens, `and` e `or` controlam o fluxo de execução: eles têm "curto-circuito" (interrompe e modificam o fluxo de execução do programa). O lado direito de um `and` não é avaliado se o lado esquerdo for `false`, e o lado direito de um `or` não é avaliado se o lado esquerdo for `true`. Em Zig, o fluxo de controle é realizado com palavras-chave, e, portanto, `and` e `or` são usados.

Além disso, o operador de comparação, `==`, não funciona entre slices, como `[]const u8`, ou seja, strings. Na maioria dos casos, você usará `std.mem.eql(u8, str1, str2)`, que comparará o comprimento e, em seguida, os bytes das duas slices.

O `if`, `else if` e `else` em Zig são comuns:

```zig
// std.mem.eql faz uma comparação byte-a-byte
// no caso de uma string, é uma comparação sensitiva
if (std.mem.eql(u8, method, "GET") or std.mem.eql(u8, method, "HEAD")) {
	// lidar com a requisição GET
} else if (std.mem.eql(u8, method, "POST")) {
	// lidar com a requisição POST
} else {
	// ...
}
```

> O primeiro argumento para `std.mem.eql` é um tipo, neste caso, `u8`. Este é o primeiro exemplo de uma função genérica que vimos. Vamos explorar isso mais detalhadamente em uma parte posterior.

O exemplo acima está comparando strings ASCII e provavelmente deveria ser insensível a maiúsculas e minúsculas. `std.ascii.eqlIgnoreCase(str1, str2)` é provavelmente uma opção melhor.

Não há operador ternário, mas você pode usar um `if/else` da seguinte forma:

```zig
const super = if (power > 9000) true else false;
```

`switch` é semelhante a um `if`/`else if`/`else`, mas tem a vantagem de ser exaustivo. Ou seja, é um erro de compilação se nem todos os casos forem tratados. Este código não será compilado:

```zig
fn anniversaryName(years_married: u16) []const u8 {
	switch (years_married) {
		1 => return "paper",
		2 => return "cotton",
		3 => return "leather",
		4 => return "flower",
		5 => return "wood",
		6 => return "sugar",
	}
}
```

Nos é dito: o `switch` deve lidar com todas as possibilidades. Como nosso `years_married` é um inteiro de 16 bits, isso significa que precisamos lidar com todos os 64 mil casos? Sim, mas felizmente há um `else`:

```zig
// ...
6 => return "sugar",
else => return "no more gifts for you",
```

Podemos combinar vários casos ou usar intervalos, e usar blocos para casos complexos:

```zig
fn arrivalTimeDesc(minutes: u16, is_late: bool) []const u8 {
	switch (minutes) {
		0 => return "arrived",
		1, 2 => return "soon",
		3...5 => return "no more than 5 minutes",
		else => {
			if (!is_late) {
				return "sorry, it'll be a while";
			}
			// a fazer, algo está muito errado
			return "never";
		},
	}
}
```

Embora um `switch` seja útil em vários casos, sua natureza exaustiva realmente se destaca ao lidar com enums, sobre as quais falaremos em breve.

O loop `for` do Zig é usado para iterar sobre arrays, slices e intervalos. Por exemplo, para verificar se um array contém um valor, poderíamos escrever:

```zig
fn contains(haystack: []const u32, needle: u32) bool {
	for (haystack) |value| {
		if (needle == value) {
			return true;
		}
	}
	return false;
}
```

Os loops `for` podem funcionar em várias sequências ao mesmo tempo, contanto que essas sequências tenham o mesmo comprimento. Acima, usamos a função `std.mem.eql`. Veja como ela (quase) se parece:

```zig
pub fn eql(comptime T: type, a: []const T, b: []const T) bool {
	// se não tiverem o mesmo comprimento, não pode ser iguais
	if (a.len != b.len) return false;

	for (a, b) |a_elem, b_elem| {
		if (a_elem != b_elem) return false;
	}

	return true;
}
```

A verificação inicial do `if` não é apenas uma otimização de desempenho agradável, é uma proteção necessária. Se a retirarmos e passarmos argumentos de comprimentos diferentes, teremos um pânico em tempo de execução: loop `for` sobre objetos com comprimentos não iguais.

Os loops `for` também podem iterar sobre intervalos, como:

```zig
for (0..10) |i| {
	std.debug.print("{d}\n", .{i});
}
```

> Nosso `switch` usou três pontos, `3...6`, enquanto este intervalo usa dois, `0..10`. Isso ocorre porque os casos do `switch` são inclusivos de ambos os números, enquanto o `for` é exclusivo do limite superior.

Isso realmente se destaca em combinação com uma (ou mais!) sequência:

```zig
fn indexOf(haystack: []const u32, needle: u32) ?usize {
	for (haystack, 0..) |value, i| {
		if (needle == value) {
			return i;
		}
	}
	return null;
}
```

> Isso é uma prévia de tipos nulos.

O final do intervalo é inferido a partir do comprimento de `haystack`, embora pudéssemos nos punir e escrever: `0..haystack.len`. Loops `for` não suportam a forma mais genérica do idioma `init; compare; step`. Para isso, contamos com o `while`.

Como o `while` é mais simples, tomando a forma de `while (condição) { }`, temos um maior controle sobre a iteração. Por exemplo, ao contar o número de sequências de escape em uma string, precisamos incrementar nosso iterador em 2 para evitar contar duas vezes uma `\\`:

```zig
var i: usize = 0;
var escape_count: usize = 0;
while (i < src.len) {
	if (src[i] == '\\') {
		i += 2;
		escape_count += 1;
	} else {
		i += 1;
	}
}
```

Um `while` pode ter uma cláusula `else`, que é executada quando a condição é falsa. Ele também aceita uma declaração para ser executada após cada iteração. Esse recurso era comumente usado antes do `for` suportar várias sequências. O exemplo acima pode ser escrito como:

```zig
var i: usize = 0;
var escape_count: usize = 0;

//                  esta parte
while (i < src.len) : (i += 1) {
	if (src[i] == '\\') {
		// +1 aqui, e +1 acima == +2
		i += 1;
		escape_count += 1;
	}
}
```

`break` e `continue` são suportados para interromper o loop mais interno ou pular para a próxima iteração.

Blocos podem ser rotulados e `break` e `continue` podem direcionar um rótulo específico. Um exemplo artificial:

```zig
outer: for (1..10) |i| {
	for (i..10) |j| {
		if (i * j > (i+i + j+j)) continue :outer;
		std.debug.print("{d} + {d} >= {d} * {d}\n", .{i+i, j+j, i, j});
	}
}
```

`break` tem outro comportamento interessante, que é o de retornar um valor de um bloco:

```zig
const personality_analysis = blk: {
	if (tea_vote > coffee_vote) break :blk "sane";
	if (tea_vote == coffee_vote) break :blk "whatever";
	if (tea_vote < coffee_vote) break :blk "dangerous";
};
```

Blocos de código como este devem ser terminados com ponto e vírgula.

Mais tarde, ao explorarmos uniões marcadas, uniões de erros e tipos opcionais, veremos o que mais essas estruturas de fluxo de controle têm a oferecer.



## Enumerações _(enums)_

Enumerações são constantes inteiras que recebem um rótulo. Eles são definidos de maneira semelhante a um struct:

```zig
// poderia ser "pub"
const Status = enum {
	ok,
	bad,
	unknown,
};
```

E, assim como um struct, pode conter outras definições, incluindo funções que podem ou não receber o enum como parâmetro:

```zig
const Stage = enum {
	validate,
	awaiting_confirmation,
	confirmed,
	completed,
	err,

	fn isComplete(self: Stage) bool {
		return self == .confirmed or self == .err;
	}
};
```

> Se você deseja a representação de string de um enum, pode usar a função embutida `@tagName(enum)`.

Lembre-se de que os tipos de struct podem ser inferidos com base no tipo atribuído ou no tipo de retorno usando a notação `.{...}`. Acima, vemos o tipo de enum sendo inferido com base em sua comparação com `self`, que é do tipo `Stage`. Poderíamos ter sido explícitos e escrito: `return self == Stage.confirmed or self == Stage.err;`. Mas, ao lidar com enums, você frequentemente verá o tipo de enum omitido via a notação `.$value`.

A natureza exaustiva do `switch` combina bem com enums, pois garante que você tratou todos os casos possíveis. Tenha cuidado ao usar a cláusula `else` de um `switch`, pois ela corresponderá a qualquer valor de enum recém-adicionado, o que pode ou não ser o comportamento desejado.



## Uniões marcadas _(tagged unions)_

Uma união define um conjunto de tipos que um valor pode ter. Por exemplo, esta união `Number` pode ser um `integer` (número inteiro), um número `float` (ponto flutuante) ou um `NaN` (não é um número):

```zig
const std = @import("std");

pub fn main() void {
	const n = Number{.int = 32};
	std.debug.print("{d}\n", .{n.int});
}

const Number = union {
	int: i64,
	float: f64,
	nan: void,
};
```

Uma união pode ter apenas um campo definido por vez; é um erro tentar acessar um campo não definido. Como definimos o campo `int`, se tentássemos acessar `n.float` em seguida, receberíamos um erro. Um de nossos campos, `nan`, tem um tipo `void`. Como definiríamos o seu valor? Utilize `{}`:

```zig
const n = Number{.nan = {}};
```

Um desafio com uniões é saber qual campo está definido. É aí que as uniões marcadas entram em jogo. Uma união marcada combina um enum com uma união, que pode ser usada em uma instrução `switch`. Considere este exemplo:

```zig
pub fn main() void {
	const ts = Timestamp{.unix = 1693278411};
	std.debug.print("{d}\n", .{ts.seconds()});
}

const TimestampType = enum {
	unix,
	datetime,
};

const Timestamp = union(TimestampType) {
	unix: i32,
	datetime: DateTime,

	const DateTime = struct {
		year: u16,
		month: u8,
		day: u8,
		hour: u8,
		minute: u8,
		second: u8,
	};

	fn seconds(self: Timestamp) u16 {
		switch (self) {
			.datetime => |dt| return dt.second,
			.unix => |ts| {
				const seconds_since_midnight: i32 = @rem(ts, 86400);
				return @intCast(@rem(seconds_since_midnight, 60));
			},
		}
	}
};
```

Observe que cada caso em nosso `switch` captura o valor tipado do campo. Ou seja, `dt` é um `Timestamp.DateTime` e `ts` é um `i32`. Esta é também a primeira vez que vemos uma estrutura aninhada dentro de outro tipo. `DateTime` poderia ter sido definido fora da união. Também estamos vendo duas novas funções embutidas: `@rem` para obter o resto e `@intCast` para converter o resultado para um `u16` (`@intCast` infere que queremos um `u16` a partir do nosso tipo de retorno, uma vez que o valor está sendo retornado).

Como podemos ver no exemplo acima, uniões marcadas podem ser usadas de alguma forma como interfaces, desde que todas as implementações possíveis sejam conhecidas antecipadamente e possam ser incorporadas na união marcada.

Finalmente, o tipo de enum de uma união marcada pode ser inferido. Em vez de definir um `TimestampType`, poderíamos ter feito:

```zig
const Timestamp = union(enum) {
	unix: i32,
	datetime: DateTime,

	...
```

e o Zig teria criado um enum implícito com base nos campos da nossa união.



## Opcionais

Qualquer valor pode ser declarado como opcional adicionando um ponto de interrogação, `?`, ao tipo. Tipos opcionais podem ser `null` ou um valor do tipo definido:

```zig
var home: ?[]const u8 = null;
var name: ?[]const u8 = "Leto";
```

A necessidade de ter um tipo explícito deve estar clara: se tivéssemos apenas feito `const name = "Leto";`, então o tipo inferido seria o não opcional `[]const u8`.

`.?` é usado para acessar o valor por trás do tipo opcional:

```zig
std.debug.print("{s}\n", .{name.?});
```

Mas teremos um pânico em tempo de execução se usarmos `.?` em um valor nulo. Uma instrução `if` pode desempacotar com segurança um valor opcional:

```zig
if (home) |h| {
	// h é um []const u8
	// temos um valor para "home"
} else {
	// não temos um valor para "home"
}
```

`orelse` pode ser usado para desempacotar o valor opcional ou executar código. Isso é comumente usado para especificar um valor padrão ou retornar da função:

```zig
const h = home orelse "unknown"
// ou talvez

// retornar nossa função
const h = home orelse return;
```

No entanto, `orelse` também pode receber um bloco e executar lógica mais complexa. Tipos opcionais também se integram com `while` e são frequentemente usados para criar iteradores. Não implementaremos um iterador aqui, mas espero que este código fictício faça sentido:

```zig
while (rows.next()) |row| {
	// realizar alguma operação com "row"
}
```



## Tipo indefinido _(undefined)_

Até agora, cada variável que vimos foi inicializada com um valor sensato. Mas às vezes não conhecemos o valor de uma variável quando ela é declarada. Tipos opcionais são uma opção, mas nem sempre fazem sentido. Nestes casos, podemos definir variáveis como `undefined` para deixá-las não inicializadas.

Um lugar comum para fazer isso é ao criar uma array que será preenchida por alguma função:

```zig
var pseudo_uuid: [16]u8 = undefined;
std.crypto.random.bytes(&pseudo_uuid);
```

O código acima ainda cria uma array de 16 bytes, mas deixa a memória não inicializada.



## Erros

Zig possui capacidades simples e pragmáticas para tratamento de erros. Tudo começa com conjuntos de erros, que se parecem e se comportam como enums:

```zig
// Assim como nosso struct na Parte 1, "OpenError" pode ser marcado como "pub"
// para torná-lo acessível do lado de fora do arquivo em que está definido
const OpenError = error {
	AccessDenied,
	NotFound,
};
```

A função, incluindo o `main`, pode agora retornar este erro:

```zig
pub fn main() void {
	return OpenError.AccessDenied;
}

const OpenError = error {
	AccessDenied,
	NotFound,
};
```

Se você tentar executar isso, receberá um erro: _"expected type 'void', found 'error{AccessDenied,NotFound}'"_. Isso faz sentido: definimos o `main` com um tipo de retorno `void`, mas estamos retornando algo (um erro, claro, mas isso ainda não é `void`). Para resolver isso, precisamos alterar o tipo de retorno de nossa função.

```zig
pub fn main() OpenError!void {
	return OpenError.AccessDenied;
}
```

Isso é chamado de tipo de união de erros e indica que nossa função pode retornar ou um erro `OpenError` ou um `void` (ou seja, nada). Até agora, fomos bastante explícitos: criamos um conjunto de erros para os possíveis erros que nossa função pode retornar e usamos esse conjunto de erros no tipo de retorno da união de erros de nossa função. No entanto, quando se trata de erros, o Zig tem alguns truques interessantes na manga. Primeiro, em vez de especificar uma união de erros como `error set!return type`, podemos deixar o Zig inferir o conjunto de erros usando: `!return type`. Portanto, poderíamos, e provavelmente iríamos, definir nosso `main` como:

```zig
pub fn main() !void
```

Ainda, Zig é capaz de criar conjuntos de erros implicitamente para nós. Em vez de criar nosso conjunto de erros, poderíamos ter feito:

```zig
pub fn main() !void {
	return error.AccessDenied;
}
```

Nossas abordagens completamente explícitas e implícitas não são exatamente equivalentes. Por exemplo, referências a funções com conjuntos de erros implícitos exigem o uso do tipo especial `anyerror`. Desenvolvedores de bibliotecas podem ver vantagens em serem mais explícitos, como código auto-documentado. Ainda assim, acredito que tanto os conjuntos de erros implícitos quanto a união de erros inferida são pragmáticos; eu faço amplo uso de ambos.

O verdadeiro valor das uniões de erros é o suporte embutido na linguagem na forma de `catch` e `try`. Uma chamada de função que retorna uma união de erros pode incluir uma cláusula `catch`. Por exemplo, uma biblioteca de servidor HTTP pode ter um código que se parece com:

```zig
action(req, res) catch |err| {
	if (err == error.BrokenPipe or err == error.ConnectionResetByPeer) {
		return;
	} else if (err == error.BodyTooBig) {
		res.status = 431;
		res.body = "Request body is too big";
	} else {
		res.status = 500;
		res.body = "Internal Server Error";
		// todo: log err
	}
};
```

A versão usando `switch` é mais idiomática:

```zig
action(req, res) catch |err| switch (err) {
	error.BrokenPipe, error.ConnectionResetByPeer) => return,
	error.BodyTooBig => {
		res.status = 431;
		res.body = "Request body is too big";
	},
	else => {
		res.status = 500;
		res.body = "Internal Server Error";
	}
};
```

Isso é tudo muito elegante, mas sejamos honestos, o mais provável é que você vai fazer em `catch` é propagar o erro para quem chamou:

```zig
action(req, res) catch |err| return err;
```

Isso é tão comum que é o que `try` faz. Em vez do exemplo acima, fazemos:

```zig
try action(req, res);
```

Isso é especialmente útil, dado que **os erros devem ser tratados**. Muito provavelmente, você fará isso com um `try` ou `catch`.

> Programadores de Go perceberão que `try` requer menos teclas do que `if err != nil { return err }`.

Na maioria das vezes, você usará `try` e `catch`, mas as uniões de erros também são suportadas por `if` e `while`, assim como os tipos opcionais. No caso de `while`, se a condição retornar um erro, a cláusula `else` será executada.

Existe um tipo especial chamado `anyerror`, que pode conter qualquer erro. Embora pudéssemos definir uma função como retornando `anyerror!TYPE` em vez de `!TYPE`, os dois não são equivalentes. O conjunto de erros inferido é criado com base no que a função pode retornar. `anyerror` é o conjunto de erros global, um superset de todos os conjuntos de erros no programa. Portanto, usar `anyerror` em uma assinatura de função provavelmente sinaliza que sua função pode retornar erros que, na realidade, ela não pode. `anyerror` é usado para parâmetros de função ou campos de estrutura que podem lidar com qualquer erro (imagine uma biblioteca de auditoria, ou _"logging"_).

Não é incomum uma função retornar uma união de erros com tipo opcional. Com um conjunto de erros inferido, isso se parece com:

```zig
// carregue o último jogo salvo
pub fn loadLast() !?Save {
	// A fazer
	return null;
}
```

Existem diferentes maneiras de consumir essas funções, mas a mais compacta é usar `try` para desembrulhar nosso erro e, em seguida, `orelse` para desembrulhar o opcional. Aqui está um esqueleto funcional:

```zig
const std = @import("std");

pub fn main() void {
	// Esta é a linha que você quer se concentrar
	const save = (try Save.loadLast()) orelse Save.blank();
	std.debug.print("{any}\n", .{save});
}

pub const Save = struct {
	lives: u8,
	level: u16,

	pub fn loadLast() !?Save {
		//a fazer
		return null;
	}

	pub fn blank() Save {
		return .{
			.lives = 3,
			.level = 1,
		};
	}
};
```

Embora o Zig tenha mais profundidade, e algumas das características da linguagem tenham capacidades mais avançadas, o que vimos nestas duas primeiras partes é uma parte significativa da linguagem. Isso servirá como uma base, permitindo-nos explorar tópicos mais complexos sem nos distrairmos muito com a sintaxe.
