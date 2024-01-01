# Memória de Pilha

Aprofundar nos ponteiros forneceu uma visão sobre a relação entre variáveis, dados e memória. Então, estamos começando a ter uma ideia de como a memória se parece, mas ainda não falamos sobre como os dados e, por extensão, a memória são gerenciados. Para scripts curtos e simples, isso provavelmente não importa. Em uma era de laptops com 32GB de RAM, você pode iniciar seu programa, usar alguns megabytes de RAM para ler um arquivo e analisar uma resposta HTTP, fazer algo incrível e sair. Na saída do programa, o sistema operacional sabe que a memória que deu ao seu programa pode agora ser usada para outra coisa.

Mas para programas que rodam por dias, meses ou até anos, a memória se torna um recurso limitado e precioso, provavelmente disputado por outros processos em execução na mesma máquina. Simplesmente não há como esperar até que o programa saia para liberar memória. Isso é o trabalho principal de um coletor de lixo: saber quais dados não estão mais em uso e liberar sua memória. No Zig, você é o coletor de lixo.

A maioria dos programas que você escreverá fará uso de três "áreas" de memória. A primeira é o espaço global, onde são armazenadas as constantes do programa, incluindo literais de string. Todos os dados globais são incorporados no binário, totalmente conhecidos em tempo de compilação (e, portanto, em tempo de execução) e imutáveis. Esses dados existem ao longo da vida do programa, nunca precisando de mais ou menos memória. Além do impacto que tem no tamanho do nosso binário, isso não é algo com que precisamos nos preocupar.

A segunda área de memória é a pilha de chamadas, o tópico desta parte. A terceira área é o heap, o tópico da nossa próxima parte.

> Não há uma diferença física real entre as áreas de memória, é um conceito criado pelo sistema operacional e pelo executável.



## Seções (quadros) da memória de pilha _(stack frames)_

Todos os dados que vimos até agora foram constantes armazenadas na seção de dados globais de nosso binário ou variáveis locais. "Local" indica que a variável é válida apenas no escopo em que é declarada. Em Zig, os escopos começam e terminam com chaves, `{ ... }`. A maioria das variáveis está limitada a uma função, incluindo os parâmetros da função, ou a um bloco de controle de fluxo, como um `if`. Mas, como vimos, você pode criar blocos arbitrários e, portanto, escopos arbitrários.

Na parte anterior, visualizamos a memória de nossas funções `main` e `levelUp`, cada uma com um `User`:

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

Há uma razão para `levelUp` estar imediatamente após `main`: esta é a nossa pilha de chamadas (simplificada). Quando nosso programa começa, `main`, junto com suas variáveis locais, é empurrado para a pilha de chamadas. Quando `levelUp` é chamado, seus parâmetros e quaisquer variáveis locais são empurrados para a pilha de chamadas. Importante, quando `levelUp` retorna, ele é retirado da pilha. Após o retorno de `levelUp` e o controle voltar para `main`, nossa pilha de chamadas parece:

```
main: user ->    -------------  (id: 1043368d0)
                 |     1     |
                 -------------  (power: 1043368d8)
                 |    100    |
                 -------------  (name.len: 1043368dc)
                 |     4     |
                 -------------  (name.ptr: 1043368e4)
                 | 1182145c0 |-------------------------
                 -------------
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

Quando uma função é chamada, todo o seu quadro de pilha é acrescentado para a pilha de chamadas. Esta é uma das razões pelas quais precisamos saber o tamanho de cada tipo. Embora possamos não saber o comprimento do nome do nosso usuário até que aquela linha específica de código seja executada (supondo que não seja uma string literal constante), sabemos que nossa função tem um `User` e, além dos outros campos, precisaremos de 8 bytes para `name.len` e 8 bytes para `name.ptr`.

Quando a função retorna, seu quadro de pilha, que foi o último empurrado para a pilha de chamadas, é retirado. Algo incrível acabou de acontecer: a memória usada por `levelUp` foi automaticamente liberada! Embora tecnicamente essa memória pudesse ser devolvida ao sistema operacional, até onde sei, nenhuma implementação realmente reduz a pilha de chamadas (elas a aumentarão dinamicamente quando necessário). Ainda assim, a memória usada para armazenar o quadro de pilha do `levelUp` está agora livre para ser usada em nosso processo para outro quadro de pilha.

> Em um programa normal, a pilha de chamadas pode ficar bastante grande. Entre todo o código de framework e bibliotecas que um programa típico utiliza, você acaba com funções profundamente aninhadas. Normalmente, isso não é um problema, mas de vez em quando, você pode encontrar algum tipo de erro de estouro de pilha, o famoso _Stack Overflow_! Isso ocorre quando nossa pilha de chamadas fica sem espaço. Mais frequentemente do que não, isso acontece com funções recursivas - uma função que chama a si mesma.

Assim como nossos dados globais, a pilha de chamadas é gerenciada pelo sistema operacional e pelo executável. No início do programa e para cada _thread_ que iniciamos posteriormente, uma pilha de chamadas é criada (cujo tamanho geralmente pode ser configurado no sistema operacional). A pilha de chamadas existe durante toda a vida do programa ou, no caso de uma thread, durante toda a vida da thread. No encerramento do programa ou da _thread_, a pilha de chamadas é liberada. Mas, ao contrário dos nossos dados globais, a pilha de chamadas contém apenas os quadros de pilha para a hierarquia de funções atualmente em execução. Isso é eficiente tanto em termos de uso de memória quanto na simplicidade de empilhar e desempilhar os quadros de pilha na pilha.



## Ponteiros pendentes _(dangling pointers)_

A pilha de chamadas é incrível tanto por sua simplicidade quanto por sua eficiência. Mas também é assustadora: quando uma função retorna, qualquer um de seus dados **locais** se torna inacessível. Isso pode parecer razoável, afinal, são dados locais, mas pode introduzir problemas sérios. Considere este código:

```zig
const std = @import("std");

pub fn main() void {
	var user1 = User.init(1, 10);
	var user2 = User.init(2, 20);

	std.debug.print("User {d} has power of {d}\n", .{user1.id, user1.power});
	std.debug.print("User {d} has power of {d}\n", .{user2.id, user2.power});
}

pub const User = struct {
	id: u64,
	power: i32,

	fn init(id: u64, power: i32) *User{
		var user = User{
			.id = id,
			.power = power,
		};
		return &user;
	}
};
```

À primeira vista, seria razoável esperar a seguinte saída:

```
User 1 has power of 10
User 2 has power of 20
```

Eu obtive:

```
User 2 has power of 20
User 9114745905793990681 has power of 0
```

Você pode obter resultados diferentes, mas com base na minha saída, `user1` herdou os valores de `user2`, e os valores de `user2` não fazem sentido. O problema-chave com este código é que `User.init` retorna o endereço do _user_ local, `&user`. Isso é chamado de ponteiro pendurado _(dangling pointer)_, um ponteiro que referencia memória inválida. Isso é a origem de muitos erros de segmentação de memória _(segfault)_.

Quando um quadro de pilha é removido da pilha de chamadas, quaisquer referências que temos a essa memória são inválidas. O resultado ao tentar acessar essa memória é indefinido. Você provavelmente obterá dados sem sentido ou um erro de segmentação. Poderíamos tentar dar algum sentido à minha saída, mas não é um comportamento no qual gostaríamos ou mesmo poderíamos confiar.

Um desafio com esse tipo de bug é que, em linguagens com coletores de lixo, o código acima é perfeitamente aceitável. Em Go, por exemplo, seria detectado que o `user` local sobrevive ao seu escopo, a função `init`, e garantiria sua validade pelo tempo que fosse necessário (como Go faz isso é um detalhe de implementação, mas tem algumas opções, incluindo mover os dados para o heap, que é sobre o que trata a próxima parte).

A outra questão, lamento dizer, é que pode ser um bug difícil de detectar. Em nosso exemplo acima, estamos claramente retornando o endereço de uma variável local. Mas tal comportamento pode se esconder dentro de funções aninhadas e tipos de dados complexos. Você vê algum possível problema com o seguinte código incompleto:

```zig
fn read() !void {
	const input = try readUserInput();
	return Parser.parse(input);
}
```

O que quer que `Parser.parse` retorne sobrevive a `input`. Se o `Parser` mantiver uma referência a `input`, isso será um ponteiro pendurado apenas esperando para causar problemas em nosso aplicativo. Idealmente, se o `Parser` precisa que `input` sobreviva tanto quanto ele, ele fará uma cópia e essa cópia estará vinculada à sua própria vida (mais sobre isso na próxima parte). Mas não há nada aqui para impor esse contrato. A documentação do `Parser` pode esclarecer o que espera de `input` ou o que faz com ele. Na falta disso, talvez seja necessário examinar o código para descobrir.

---

A maneira simples de resolver nosso bug inicial é modificar `init` para que retorne um `User` em vez de um `*User` (ponteiro para `User`). Então, seria possível simplesmente utilizar `return user;` em vez de `return &user;`. Mas nem sempre será possível fazer isso. Frequentemente, os dados precisam existir além dos limites rígidos dos escopos de funções. Para isso, temos a terceira área de memória, o heap, que será abordada na próxima parte.

Antes de entrar no heap, saiba que veremos um exemplo final de ponteiros pendurados antes do final deste guia. Nesse ponto, teremos coberto o suficiente da linguagem para apresentar um exemplo um pouco menos confuso. Quero revisitar esse tópico porque, para os desenvolvedores que vêm de linguagens com coleta de lixo, isso provavelmente causará bugs e frustração. No entanto, é algo que você vai dominar. Tudo se resume a estar ciente de onde e quando os dados existem na memória.
