# Estilização

Nesta breve parte, abordaremos duas regras de codificação impostas pelo compilador, bem como a convenção de nomenclatura da biblioteca padrão.



## Variáveis não utilizadas

Zig não permite que variáveis fiquem sem uso. O seguinte código resultará em dois erros de compilação:

```zig
const std = @import("std");

pub fn main() void {
	const sum = add(8999, 2);
}

fn add(a: i64, b: i64) i64 {
	// atenção ao fato que isto é a + a, não a + b
	return a + a;
}
```

O primeiro erro ocorre porque `sum` é uma constante local não utilizada _(unused local constant)_. O segundo erro ocorre porque `b` é um parâmetro de função não utilizado _(unused function parameter)_. Para este código, esses são bugs óbvios. No entanto, você pode ter razões legítimas para ter variáveis e parâmetros de função não utilizados. Nestes casos, você pode atribuir as variáveis ao sublinhado (_):

```zig
const std = @import("std");

pub fn main() void {
	_ = add(8999, 2);

	// ou

	sum = add(8999, 2);
	_ = sum;
}

fn add(a: i64, b: i64) i64 {
	_ = b;
	return a + a;
}
```

Como alternativa a fazer `_ = b;`, poderíamos ter nomeado o parâmetro da função como `_`, embora, na minha opinião, isso deixe o leitor adivinhando qual é o parâmetro não utilizado:

```zig
fn add(a: i64, _: i64) i64 {
```

Observe que `std` também está sem uso, mas não gera um erro. Em algum momento no futuro, espera-se que o Zig também trate isso como um erro de compilação.



## Sombreamento

Zig não permite que um identificador "oculte" outro usando o mesmo nome. Este código, para ler de um _socket_, não é válido:

```zig
fn read(stream: std.net.Stream) ![]const u8 {
	var buf: [512]u8 = undefined;
	const read = try stream.read(&buf);
	if (read == 0) {
		return error.Closed;
	}
	return buf[0..read];
}
```

A nossa variável `read` oculta o nome da nossa função. Eu não sou fã dessa regra, pois geralmente leva os desenvolvedores a usarem nomes curtos e sem significado. Por exemplo, para fazer com que este código compile, eu mudaria `read` para `n`. Esta é uma situação em que, na minha opinião, os desenvolvedores estão em uma posição muito melhor para escolher a opção mais legível.



## Convenções de Nomenclatura

Além das regras impostas pelo compilador, você é, obviamente, livre para seguir qualquer convenção de nomenclatura que preferir. Mas ajuda a entender a convenção de nomenclatura do próprio Zig, já que grande parte do código com o qual você irá interagir, desde a biblioteca padrão até bibliotecas de terceiros, nos torna parte dela.

O código-fonte Zig é recuado com 4 espaços. Eu pessoalmente uso um tab porque é melhor para acessibilidade.

Os nomes das funções são _camelCase_ e as variáveis são _snake_case_. Os tipos são _PascalCase_. Há uma intersecção interessante entre essas três regras. Variáveis que fazem referência a um tipo, ou funções que retornam um tipo, seguem a regra de tipo e são _PascalCase_. Já vimos isso, embora você possa não ter percebido.

```zig
std.debug.print("{any}\n", .{@TypeOf(.{.year = 2023, .month = 8})});
```

Vimos outras funções integradas: `@import`, `@rem` e `@intCast`. Como essas são funções, elas são _camelCase_. `@TypeOf` também é uma função interna, mas é _PascalCase_, por quê? Porque retorna um tipo e, portanto, a convenção de nomenclatura de tipo é usada. Se atribuíssemos o resultado de `@TypeOf` a uma variável, usando a convenção de nomenclatura do Zig, essa variável também deveria ser _PascalCase_:

```zig
const T = @TypeOf(3)
std.debug.print("{any}\n", .{T});
```

O executável `zig` possui um comando `fmt` que, dado um arquivo ou diretório, formatará o arquivo com base no guia de estilo do próprio Zig. Porém, ele não cobre tudo, por exemplo, ajustará o recuo e as posições dos colchetes, mas não irá alternar entre identificados maiúsculos e minúsculos.
