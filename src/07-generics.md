# Genéricos (parametrização polimórfica)

No final da parte anterior, construímos uma matriz dinâmica básica chamada `IntList`. O objetivo dessa estrutura de dados era armazenar um número dinâmico de valores. Embora o algoritmo que usamos funcionasse para qualquer tipo de dado, nossa implementação estava vinculada a valores `i64`. Aí entram os genéricos, cujo objetivo é abstrair algoritmos e estruturas de dados de tipos específicos.

Muitas linguagens implementam genéricos com sintaxe especial e regras específicas para genéricos. No caso do Zig, os genéricos são menos uma característica específica e mais uma expressão do que a linguagem é capaz. Especificamente, os genéricos aproveitam a poderosa metaprogramação em tempo de compilação do Zig.

Vamos começar olhando para um exemplo bobo, apenas para nos situarmos:

```zig
const std = @import("std");

pub fn main() !void {
	var arr: IntArray(3) = undefined;
	arr[0] = 1;
	arr[1] = 10;
	arr[2] = 100;
	std.debug.print("{any}\n", .{arr});
}

fn IntArray(comptime length: usize) type {
	return [length]i64;
}
```

O código acima imprime `{ 1, 10, 100 }`. A parte interessante é que temos uma função que retorna um `type` (portanto, a função tem _PascalCase_). E não é qualquer tipo, mas um tipo com base em um parâmetro de função. Esse código só funcionou porque declaramos `length` como `comptime`. Ou seja, exigimos que quem chama `IntArray` forneça um parâmetro de comprimento conhecido em tempo de compilação. Isso é necessário porque nossa função retorna um tipo (`type`) e os tipos (`types`) devem sempre ser conhecidos em tempo de compilação.

Uma função pode retornar qualquer tipo, não apenas primitivos e arrays. Por exemplo, com uma pequena alteração, podemos fazê-la retornar uma estrutura:

```zig
const std = @import("std");

pub fn main() !void {
	var arr: IntArray(3) = undefined;
	arr.items[0] = 1;
	arr.items[1] = 10;
	arr.items[2] = 100;
	std.debug.print("{any}\n", .{arr.items});
}

fn IntArray(comptime length: usize) type {
	return struct {
		items: [length]i64,
	};
}
```

Pode parecer estranho, mas o tipo de `arr` realmente é `IntArray(3)`. É um tipo como qualquer outro tipo, e `arr` é um valor como qualquer outro valor. Se chamássemos `IntArray(7)`, seria um tipo diferente. Talvez possamos organizar melhor as coisas:

```zig
const std = @import("std");

pub fn main() !void {
	var arr = IntArray(3).init();
	arr.items[0] = 1;
	arr.items[1] = 10;
	arr.items[2] = 100;
	std.debug.print("{any}\n", .{arr.items});
}

fn IntArray(comptime length: usize) type {
	return struct {
		items: [length]i64,

		fn init() IntArray(length) {
			return .{
				.items = undefined,
			};
		}
	};
}
```

À primeira vista, pode não parecer mais organizado. Mas além de não ter nome e estar aninhada em uma função, nossa estrutura está se parecendo com qualquer outra estrutura que vimos até agora. Ela tem campos, tem funções. Você sabe o que dizem: se parece com um pato.... Bem, isso se parece, nada e grasna como uma estrutura normal, porque é.

Tomamos esse caminho para nos familiarizarmos com uma função que retorna um tipo e a sintaxe correspondente. Para obter um genérico mais típico, precisamos fazer uma última alteração: nossa função precisa receber um tipo. Na realidade, esta é uma pequena mudança, mas `type` pode parecer mais abstrato do que `usize`, então fizemos isso lentamente. Vamos dar um salto e modificar nossa `IntList` anterior para funcionar com qualquer tipo. Começaremos com um esqueleto:

```zig
fn List(comptime T: type) type {
	return struct {
		pos: usize,
		items: []T,
		allocator: Allocator,

		fn init(allocator: Allocator) !List(T) {
			return .{
				.pos = 0,
				.allocator = allocator,
				.items = try allocator.alloc(T, 4),
			};
		}
	}
};
```

A estrutura (`struct`) acima é quase idêntica à nossa `IntList` anterior, exceto que `i64` foi substituído por `T`. Esse `T` pode parecer especial, mas é apenas um nome de variável. Poderíamos tê-lo chamado de `item_type`. No entanto, seguindo a convenção de nomenclatura do Zig, variáveis do tipo `type` são escritas em _PascalCase_.

> Para o bem ou para o mal, usar uma única letra para representar um parâmetro de tipo é muito mais antigo que o Zig. `T` é um padrão comum na maioria das linguagens, mas você verá variações específicas do contexto, como mapas de hash usando `K` e `V` para seus tipos de parâmetros de chave e valor.

Se você não tem certeza sobre nosso esqueleto, considere os dois lugares onde usamos `T`: `items: []T` e `allocator.alloc(T, 4)`. Quando queremos usar esse tipo genérico, criaremos uma instância usando:

```zig
var list = try List(u32).init(allocator);
```

Quando o código é compilado, o compilador cria um novo tipo substituindo cada `T` por `u32`. Se usarmos `List(u32)` novamente, o compilador reutilizará o tipo que foi criado anteriormente. Se especificarmos um novo valor para `T`, como `List(bool)` ou `List(User)`, novos tipos serão criados.

Para completar nossa Lista genérica, podemos literalmente copiar e colar o restante do código do `IntList` e substituir `i64` por `T`. Aqui está um exemplo completo funcional:

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

pub fn main() !void {
	var gpa = std.heap.GeneralPurposeAllocator(.{}){};
	const allocator = gpa.allocator();

	var list = try List(u32).init(allocator);
	defer list.deinit();

	for (0..10) |i| {
		try list.add(@intCast(i));
	}

	std.debug.print("{any}\n", .{list.items[0..list.pos]});
}

fn List(comptime T: type) type {
	return struct {
		pos: usize,
		items: []T,
		allocator: Allocator,

		fn init(allocator: Allocator) !List(T) {
			return .{
				.pos = 0,
				.allocator = allocator,
				.items = try allocator.alloc(T, 4),
			};
		}

		fn deinit(self: List(T)) void {
			self.allocator.free(self.items);
		}

		fn add(self: *List(T), value: T) !void {
			const pos = self.pos;
			const len = self.items.len;

			if (pos == len) {
			    // ficamos sem espaço
			    // cria-se um "slice" duas vezes maior
				var larger = try self.allocator.alloc(T, len * 2);

                // copia-se os itens que adicionamos previamente ao nosso novo espaço criado
				@memcpy(larger[0..len], self.items);

				self.allocator.free(self.items);

				self.items = larger;
			}

			self.items[pos] = value;
			self.pos = pos + 1;
		}
	};
}
```

Nossa função `init` retorna uma `List(T)`, e nossas funções `deinit` e `add` recebem uma `List(T)` e `*List(T)`. Para nossa classe simples, isso está bem, mas para estruturas de dados grandes, escrever o nome genérico completo pode se tornar um pouco tedioso, especialmente se tivermos múltiplos parâmetros de tipo (por exemplo, um mapa hash que aceita um tipo separado para sua chave e valor). A função embutida `@This()` retorna o tipo mais interno de onde é chamada. Muito provavelmente, nosso `List(T)` seria escrito como:

```zig
fn List(comptime T: type) type {
	return struct {
		pos: usize,
		items: []T,
		allocator: Allocator,

		// Adicionado
		const Self = @This();

		fn init(allocator: Allocator) !Self {
			// mesmo código
		}

		fn deinit(self: Self) void {
			// mesmo código
		}

		fn add(self: *Self, value: T) !void {
			// mesmo código
		}
	};
}
```

`Self` não é um nome especial, é apenas uma variável, e está em _PascalCase_ porque seu valor é um tipo (`type`). Podemos usar `Self` onde anteriormente usávamos `List(T)`.



---

Poderíamos criar exemplos mais complexos, com vários parâmetros de tipo e algoritmos mais avançados. No entanto, no final, o código genérico essencial não seria diferente dos exemplos simples acima. Na próxima parte, voltaremos a abordar os genéricos ao analisar a `ArrayList(T)` e `StringHashMap(V)` da biblioteca padrão.
