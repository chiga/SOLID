# SOLID

O SOLID é um acrônimo criado por criado por Michael Feathers a partir dos cinco primeiros princípiocs ("five first principles") da programação orientada à objetos identificados por Robert C. Martin no início dos anos 2000.

Esses princípios tem como objetivo obter as vantagens da orientação a objetos através de um código de alta qualidade e permita:
- Leitura, testes e manutenção fáceis
- Extensibilidade com o menor esforço possível
- Reaproveitamento
- Maximização do tempo de utilização do código

A aplicação correta dos princípios SOLID permite um código altamente estruturado e desacoplado de modo que são evitadas, devido à uma estrutura frágil:
- Dificuldades no teste
- Dificuldades no isolamento de funcionalidades
- Duplicação de código
- Quebra de código em vários lugares após alterações

A seguir vamos estudar cada um dos 5 príncipios que compõem o SOLID.

[Single Responsability Principle](#single-responsability-principle)

[Open-Closed Principle](#open-closed-principle)

[Liskov Substitution Principle](#liskov-substitution-principle)

[Interface Segregation Principle](#interface-segregation-principle)

[Dependency Inversion Principle](#dependency-inversion-principle)

## Single Responsability Principle

> "A class should have one, and only one, reason to change"

O Princípio da Responsabilidade Única (Single Responsability Principle) diz que "uma classe deve ter um único motivo para ser modificada". Em outras palavras, uma classe deve ter uma única responsabilidade, um único motivo de existir.

Veja o exemplo da classe Usuário abaixo.

### Violação do SRP

```c#
public class Usuario
{
    public int Id { get; set; }
    public string Nome { get; set; }
    public string Email { get; set; }
    public string CPF { get; set; }

    public Tuple<bool, string> InserirUsuario()
    {
        if (CPF.Length != 11)
            return new Tuple<bool, string>(false, "O CPF deve ter 11 caracteres");

        using (var connection = new SqlConnection())
        {
            var command = new SqlCommand();

            connection.ConnectionString = "Local";

            command.Connection = connection;
            command.CommandType = System.Data.CommandType.Text;
            command.CommandText = $"INSERT INTO USUARIO (ID, NOME, EMAIL, CPF) VALUES (@id, @nome, @email, @cpf)";

            command.Parameters.AddWithValue("id", Id);
            command.Parameters.AddWithValue("nome", Nome);
            command.Parameters.AddWithValue("email", Email);
            command.Parameters.AddWithValue("cpf", CPF);

            connection.Open();
            command.ExecuteNonQuery();
        }

        return new Tuple<bool, string>(true, "Usuário inserido com sucesso");
    }
}
```

Da maneira como foi implantada, essa classe viola o SRP uma vez que uma classe não deve delegar responsabilidades à ela própria. No caso do exemplo, a classe não deve validar a si mesma ou se adicionar ao banco. Outro exemplo de violação está no fato de que um método de Inserção de um determinado registro não deve ter a responsabilidade de validar esse registro.

Logo, podemos concluir que uma classe construída da maneira demonstrada acima possui múltiplas responsabilidades, o que vai contra o SRP.

### SRP da maneira correta

Para apicar o SRP de uma maneira correta então, devemos separar as responsabilidades descritas acima, de modo que haja exclusivamente uma classe para cada responsabilidade:

- Manter as propriedades e indicar se estas são válidas ou não.

```c#
public class Usuario
{
    public int Id { get; set; }
    public string Nome { get; set; }
    public string Email { get; set; }
    public string CPF { get; set; }
    public bool IsValid { get { return UsuarioValidation.Validar(); } }
}
```

- Executar essa validação.

```c#
public static class UsuarioValidation
{
    public bool Validar(Usuario usuario)
    {
        return usuario != null && usuario.CPF != null && usuario.CPF != String.Empty && usuario.CPF.Length == 11;
    }
}
```

- Interagir com o Banco.

```c#
public class UsuarioRepository
{
    public bool Adicionar(Usuario usuario)
    {
        try
        {
            using (var connection = new SqlConnection())
            {
                var command = new SqlCommand();

                connection.ConnectionString = "Local";
                command.Connection = connection;
                command.CommandType = System.Data.CommandType.Text;
                command.CommandText = $"INSERT INTO USUARIO (ID, NOME, EMAIL, CPF) VALUES (@id, @nome, @email, @cpf)";

                command.Parameters.AddWithValue("id", Id);
                command.Parameters.AddWithValue("nome", Nome);
                command.Parameters.AddWithValue("email", Email);
                command.Parameters.AddWithValue("cpf", CPF);

                connection.Open();
                command.ExecuteNonQuery();
            }

            return true;
        }
        catch (Exception)
        {
            return false;
        }
    }
}
```

- Executar o processo de adição de um novo Usuário.
```c#
public class UsuarioService
{
    public bool Inserir(Usuario usuario)
    {
        if (usuario.IsValid())
            return false;

        var repository = new UsuarioRepository();
        repository.Adicionar(usuario);
    }
}
```

De uma maneira bem grosseira, corrigimos a violação do SRP separando uma única classe Usuário, que antes era responsável por verificar se suas propriedades eram válidas, criar uma conexão com o banco e efetuar a inserção, em 4 classes com responsabilidades únicas. Dessa forma, desacoplamos o código, facilitando alterações e testes do mesmo. 

Imagine, por exemplo, que os seus dados hoje guardados em um banco SQL passem a ser guardados em um arquivo XML. A única alteração a ser feita será no Repositório, mantendo todo o resto do nosso código intacto, uma vez que o Serviço e a Validação não precisam saber de que maneira é feita a conexão e a inserção na base de dados.

## Open-Closed Principle

> "Software entities (classes, modules, functions, etc.) should be open for extension but closed for modification"

Esse princípio permite que o código seja extendido sem se preocupar com as classes, métodos legados. Uma vez que estas classes e métodos foram criados, eles não devem ser mais modificados, mas sim devem estar abertas para extensão.

### Violação do OCP

Seja o seguinte exemplo: um serviço de impressão de notas fiscais em um sistema de controle de estoque. O estoque é responsável por receber as mercadorias e devolvê-las caso elas tenham algum problema, por exemplo.

```c#
public class NotaFiscal { }

public class NotaDeRecebimentoDeMercadoria : NotaFiscal
{
	public void GerarNotaDeRecebimento() { } 
}

public class NotaDeDevolucaoDeMercadoria : NotaFiscal
{
	public void GerarNotaDeDevolucao() { }
}

public class ImpressaoDeNotas()
{
	public Imprimirnotas(IEnumerable<NotaFiscal> notas)
	{
		foreach(var notaFiscal in notas)
		{
			if(notaFiscal is NotaDeRecebimentoDeMercadoria)
				(notaFiscal as NotaDeRecebimentoDeMercadoria).GerarNotaDeRecebimento();
			else if(notaFiscal is NotaDeDevolucaoDeMercadoria)
				(notaFiscal as NotaDeDevolucaoDeMercadoria).GerarNotaDeDevolucao();
		}
	}
}
```

Suponha que agora o nosso serviço precise contemplar um novo tipo de nota fiscal como uma nota de saída de mercadoria, emitida quando alguma mercadoria sai para entrega, por exemplo. Seguindo a mesma lógica implementada acima, o código ficaria assim:

```c#
public class NotaFiscal { }

public class NotaDeRecebimentoDeMercadoria : NotaFiscal
{
	public void GerarNotaDeRecebimento() { } 
}

public class NotaDeDevolucaoDeMercadoria : NotaFiscal
{
	public void GerarNotaDeDevolucao() { }
}

public class NotaDeSaidaDeMercadoria : NotaFiscal
{
	public void GerarNotaDeSaida() { }
}

public class ImpressaoDeNotas()
{
	public Imprimirnotas(IEnumerable<NotaFiscal> notas)
	{
		foreach(var notaFiscal in notas)
		{
			if(notaFiscal is NotaDeRecebimentoDeMercadoria)
				(notaFiscal as NotaDeRecebimentoDeMercadoria).GerarNotaDeRecebimento();
			else if(notaFiscal is NotaDeDevolucaoDeMercadoria)
				(notaFiscal as NotaDeDevolucaoDeMercadoria).GerarNotaDeDevolucao();
			else if(notaFiscal is NotaDeSaidaDeMercadoria)
				(notaFiscal as NotaDeSaidaDeMercadoria).GerarNotaDeSaida();
		}
	}
}
```
Além de criar uma nova classe e implementar um novo método, a classe de ImpressaoDeNotas teria que ser alterada, mais precisamente o método Imprimir notas teria que ser alterado, para contemplar a impressão do novo tipo de Nota Fiscal. Daí, pode-se concluir que esse método não segue o OCP.

### OCP da maneira correta

Para implementar o OCP corretamente, algumas coisas devem ser mudadas. Valendo-se de coneceitos como herança e polimorfismo, é interessante tornar classe NotaFiscal abstrata e implementar nela um método abstrato GerarNotaFiscal que será sobreescrito pelas classes filhas, garantindo assim que as classes filhas tenham esse método com a mesma chamada, mas com implementações diferentes.

```c#
public abstract class NotaFiscal 
{
	public abstract void GerarNotaFiscal()
}

public class NotaDeRecebimentoDeMercadoria : NotaFiscal
{
	public override GerarNotaFiscal() { }
}

public class NotaDeDevolucaoDeMercadoria : NotaFiscal
{
	public override GerarNotaFiscal() { }
}

public class NotaDeSaidaDeMercadoria : NotaFiscal
{
	public override GerarNotaFiscal() { }
}

public class ImpressaoDeNotas()
{
	public Imprimirnotas(IEnumerable<NotaFiscal> notas)
	{
		foreach(var notaFiscal in notas)
		{
			notaFiscal.GerarNotaFiscal();
		}
	}
}
```

Desse modo, nosso serviço de impressão de notas está aberto para imprimir novos tipos de notas que possam surgir no controle do estoque, mas fechado para modificações.

## Liskov Substitution Principle

> "Let q(x) be a property provable of objects x of a type T. Then q(y) should be provable objects of type S, where S is a subtype of T."

Em outras palavras, "uma classe base deve poder ser substituída por sua classe derivada". Esse princípio foi proposto por Barabara Liskov em um artigo científico no ano de 1988.

O OCP, visto acima, trabalha em conjunto com este princípio, garantindo sua aplicação, uma vez que o OCP induz à utilização abstrações e heranças. Contudo, é necessário ter cuidado para saber se a abstração aplicada é uma abstração válida. Caso não seja, temos uma violação do LSP.

### Violação do LSP

Um exemplo clássico de violação do LSP seria o seguinte

```c#

```

## Interface Segregation Principle

## Dependency Inversion Principle
