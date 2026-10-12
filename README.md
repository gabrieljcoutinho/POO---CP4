# Checkpoint 4 — Bug Hunt StreamFIAP

## Identificação

**Grupo:** cp4-bughunt-debugadores

| Integrante | RM | Turma |
|---|---|---|
| Gabriel Jorge Coutinho | 565441 |2CCPW|

| Campo | |
|---|---|
| **Total de bugs corrigidos** | 12 / 12 |
| **Total de ajustes de Clean Code** | 6 / 6 |

---

## Parte 1 — Bugs encontrados

| # | Sintoma observado (o que fiz/vi) | Causa raiz (arquivo e linha aproximada) | Correção aplicada | Conceito da disciplina |
|---|---|---|---|---|
| bug01 | `GET /{id}/preco-promocional` retornava valor maior que o preço normal para Filme — a "promoção" aumentava o preço em 20% em vez de diminuir | `Filme.java` — `aplicarPromocao()` usava `preco * 1.2` (multiplicador de acréscimo) | Alterado para `preco * 0.8`, aplicando o desconto de 20% conforme o contrato | Interface `Promocionavel` e polimorfismo — o contrato da interface exige 20% de desconto; a implementação violava esse contrato |
| bug02 | Ao cadastrar uma série (`POST /api/conteudos/serie`), o conteúdo era salvo sem título, categoria, duração e classificação etária (campos nulos/zero) | `Serie.java` — construtor com 5 parâmetros não chamava `super(...)`, então os campos herdados de `Conteudo` nunca eram inicializados | Adicionado `super(titulo, categoria, duracaoMinutos, classificacaoEtaria, true)` no construtor de `Serie` | Herança e construtores — subclasses devem chamar o construtor da superclasse para inicializar os atributos herdados |
| bug03 | Série sempre cobrava R$9,90 independentemente do número de temporadas; o método `calcularPrecoAluguel` de `Serie` nunca era chamado | `Serie.java` — o método tinha assinatura `calcularPrecoAluguel(double desconto)` (parâmetro extra), criando uma sobrecarga em vez de sobrescrever o método `calcularPrecoAluguel()` da superclasse | Removido o parâmetro, corrigida a assinatura para `calcularPrecoAluguel()` e adicionado `@Override` | Sobrescrita (override) vs sobrecarga (overload) — o compilador aceita as duas, mas sem `@Override` a sobrecarga silenciosa faz o polimorfismo falhar |
| bug04 | Usuário com 0 créditos conseguia alugar qualquer conteúdo sem erro; créditos ficavam negativos | `Usuario.java` — `temCreditosSuficientes()` comparava `preco >= this.creditos`, que é verdadeiro quando o usuário não tem saldo, lógica completamente invertida | Corrigido para `this.creditos >= preco` | Encapsulamento e validação de regra de negócio — método que guarda invariante de negócio continha lógica booleana invertida |
| bug05 | Usuário era cadastrado com nome `null` no banco, mesmo enviando o nome no JSON | `Usuario.java` — construtor fazia `nome = nome` (atribuindo o parâmetro a si mesmo) em vez de `this.nome = nome`, deixando o campo de instância sem valor | Corrigido para `this.nome = nome` | Encapsulamento e construtores — ausência do `this` em atribuição de campo com mesmo nome do parâmetro cria bug silencioso |
| bug06 | `POST /api/usuarios` retornava erro ou salvava usuário sem ID gerado; não era possível buscar o usuário criado por ID | `Usuario.java` — anotação `@Id` sem `@GeneratedValue`, então o JPA não sabia como gerar o identificador automaticamente | Adicionado `@GeneratedValue(strategy = GenerationType.IDENTITY)` sobre o campo `id` | Persistência com JPA/Spring Data — sem `@GeneratedValue` o JPA exige que a aplicação forneça o ID manualmente |
| bug07 | `GET /api/conteudos/999` retornava HTTP 200 com corpo vazio em vez de 404 com mensagem de erro | `ConteudoController.java` — bloco `try/catch (Exception e)` com corpo vazio engolia a `ConteudoNaoEncontradoException`, e o método retornava `null` (serializado como 200 vazio) | Removido o bloco try/catch; a exceção propaga naturalmente para o `GlobalExceptionHandler`, que retorna 404 | Exceções — catch vazio é antipadrão grave; impede que o handler global intercepte a exceção e devolva a resposta correta ao cliente |
| bug08 | `GET /api/conteudos/categoria/FICCAO` sempre retornava lista vazia, mesmo com conteúdos cadastrados nessa categoria | `ConteudoController.java` — comparação `c.getCategoria() == categoria` usa `==` em Strings, comparando referências de memória em vez de conteúdo; strings vindas do banco têm referência diferente das strings do path variable | Substituído por `categoria.equalsIgnoreCase(c.getCategoria())` | Strings em Java — `==` compara referência, `.equals()` compara valor; erro clássico em Java para quem vem de outras linguagens |
| bug09 | Conteúdo marcado como `disponivel: false` era alugado normalmente; o campo `ConteudoIndisponivelException` existia mas nunca era lançado | `Usuario.java` — método `alugar()` não verificava `c.isDisponivel()` antes de prosseguir | Adicionado bloco `if (!c.isDisponivel())` no início de `alugar()`, lançando `ConteudoIndisponivelException` com mensagem adequada | Regras de negócio no model — a validação de disponibilidade é responsabilidade do model, não do controller |
| bug10 | Conteúdos com `duracaoMinutos: -10` ou `0` eram aceitos e salvos no banco sem qualquer erro | `ConteudoController.java` — métodos `cadastrarFilme`, `cadastrarSerie` e `cadastrarDocumentario` não validavam o campo `duracaoMinutos` | Adicionada verificação `if (filme.getDuracaoMinutos() <= 0)` em cada método de cadastro, retornando HTTP 400 com mensagem de erro | Validação de entrada — dados vindos do cliente (boundary da aplicação) devem ser validados antes de persistir |
| bug11 | Ao tentar alugar conteúdo para usuário abaixo da classificação etária, a API retornava HTTP 500 genérico sem mensagem útil | `ClassificacaoIndicativaException.java` estendia `Exception` (checked), e o `GlobalExceptionHandler` não tinha handler para ela; o Spring não sabia traduzir essa exceção para uma resposta HTTP | Alterado para `extends RuntimeException` e adicionado `@ExceptionHandler(ClassificacaoIndicativaException.class)` no `GlobalExceptionHandler` retornando HTTP 403 com a mensagem da exceção | Exceções checked vs unchecked — checked exige tratamento explícito em cada chamador; unchecked propaga naturalmente até o handler global do Spring |
| bug12 | `GET /{id}/preco-promocional` e o fluxo de aluguel cobravam R$9,90 por um documentário, que deveria ser gratuito (R$0,00) | `Documentario.java` — não sobrescrevia `calcularPrecoAluguel()`, herdando o valor padrão `return 9.90` definido em `Conteudo` | Adicionado `@Override public double calcularPrecoAluguel() { return 0.0; }` em `Documentario` | Polimorfismo e sobrescrita — sem `@Override`, a subclasse herda o comportamento da superclasse; cada tipo de conteúdo deve definir seu próprio preço |

---

## Parte 2 — Ajustes de Clean Code

| # | Onde estava | Qual princípio/boas práticas era violado | O que eu mudei |
|---|---|---|---|
| clean01 | `Conteudo.java` — campo `public int duracaoMinutos` | Encapsulamento: atributo de instância exposto como `public` permite que qualquer classe leia e altere o valor diretamente, sem passar pelo setter | Alterado para `private int duracaoMinutos`; o getter e setter já existiam e continuam funcionando normalmente |
| clean02 | `ConteudoController.java` — método privado `calcularDescontoAntigo()` no final da classe | Código morto: método nunca chamado por nenhuma parte do sistema ocupa espaço, confunde leitores e pode mascarar lógica obsoleta | Método completamente removido |
| clean03 | `Usuario.java` — comentário `// adiciona o valor aos créditos do usuário` dentro de `debitarCreditos()` | Comentário enganoso: o comentário dizia o oposto do que o método faz (ele debita, não adiciona), violando a regra de que comentários não devem contradizer o código | Comentário removido; o nome do método já comunica o comportamento com clareza |
| clean04 | `ConteudoController.java` — comentários acima de cada método (`// GET /api/conteudos - Listar todos`, `// POST /api/conteudos/filme - cadastra um filme`, etc.) | Comentários que descrevem o quê: a anotação `@GetMapping`, `@PostMapping` e o nome do método já expressam rota e intenção; o comentário repete a informação sem agregar nada | Todos os comentários de cabeçalho de método removidos do controller |
| clean05 | `Usuario.java` — bloco de oito `System.out.println` dentro do método `alugar()` | Logging com saída padrão: em produção, `System.out.println` não é configurável, não tem nível de severidade e mistura saída de negócio com logs do servidor; viola a separação de responsabilidades | Bloco de `System.out.println` completamente removido |
| clean06 | `AluguelController.java` e `UsuarioController.java` — comentários `// POST /api/alugueis?usuarioId=1...` e `// GET /api/usuarios/{id} - Buscar usuário por ID` | Comentários redundantes: repetem informação já visível nas anotações Spring (`@PostMapping`, `@GetMapping`) e nos nomes dos métodos (`alugar`, `buscarPorId`), poluindo o código sem acrescentar contexto | Comentários removidos dos dois controllers |

---

## Parte 3 — Perguntas de reflexão

### 1. Injeção de dependência (Aula 13)

Em `ConteudoController`, `UsuarioController` e `AluguelController`, os repositories chegam via `@Autowired`, não por `new`. Se usássemos `new ConteudoRepository()`, teríamos um problema: `ConteudoRepository` é uma interface — ela não tem construtor. O Spring Data JPA gera em tempo de execução uma classe concreta que implementa toda a lógica de CRUD. O Spring gerencia o ciclo de vida desse objeto (criação, configuração de conexão com banco, singleton) e o injeta onde for necessário. Com `new`, nós mesmos teríamos que instanciar a implementação concreta, configurar a conexão manualmente e garantir que só existisse uma instância — perdendo todos os benefícios do container IoC. A injeção também facilita testes, pois permite substituir o repository por um mock sem mudar o controller.

### 2. JDBC vs Spring Data JPA (Aulas 12 e 13)

Na Aula 12 escrevemos um `ProdutoDAO` com `Connection`, `PreparedStatement` e `ResultSet`: abrir conexão, montar SQL, executar, mapear coluna por coluna para objeto, fechar conexão — dezenas de linhas para um único `findById`. Aqui, `ConteudoRepository` tem apenas a declaração da interface estendendo `JpaRepository`, e o Spring Data JPA gera automaticamente todo esse código: `findAll()`, `findById()`, `save()`, `delete()`. Além disso, o método `findByCategoria(String categoria)` funciona sem implementação alguma — o Spring interpreta o nome do método e gera o SQL `WHERE categoria = ?`. O JDBC ainda é superior quando precisamos de consultas muito específicas com JOINs complexos, controle total do SQL ou performance crítica, pois o JPA pode gerar queries ineficientes sem que percebamos.

### 3. Exceções checked vs unchecked (Aula 11)

`ClassificacaoIndicativaException` originalmente estendia `Exception` (checked). Isso obrigava todo método que a lançasse a declarar `throws ClassificacaoIndicativaException` na assinatura — o `AluguelController` e `Usuario.alugar()` tinham essa declaração. O problema: o `GlobalExceptionHandler` do Spring captura exceções via reflexão em tempo de execução, mas a `ClassificacaoIndicativaException` não era `RuntimeException`, então quando ela estourava, o Spring a tratava como erro não mapeado e retornava HTTP 500. Ao mudar para `extends RuntimeException` (unchecked), a exceção propaga naturalmente pela pilha de chamadas sem precisar ser declarada, e o handler `@ExceptionHandler(ClassificacaoIndicativaException.class)` intercepta e retorna HTTP 403 com a mensagem `"Usuário de X anos não pode assistir a Y (classificação Z anos)"`, tornando o erro compreensível para o cliente da API.

### 4. Sobrescrita vs sobrecarga (Aula 7)

Em `Serie.java`, o método estava declarado como `calcularPrecoAluguel(double desconto)` — com um parâmetro `double` que não existia na superclasse. O compilador não reclamou porque estava criando um método novo (sobrecarga), não substituindo o existente. Quando o `AluguelController` chamava `conteudo.calcularPrecoAluguel()` em um objeto `Serie`, o Java usava o método da superclasse `Conteudo`, que retorna fixo `9.90`. O método correto da `Serie` nunca era invocado. A anotação `@Override` teria impedido o bug: se a assinatura não bate exatamente com nenhum método da superclasse, o compilador recusa com erro de compilação — o problema teria sido detectado antes de qualquer teste. Após a correção, `@Override public double calcularPrecoAluguel()` garante que o polimorfismo funcione: a chamada `conteudo.calcularPrecoAluguel()` em um `Serie` executa `4.90 * numeroTemporadas`.

### 5. Onde blindar o objeto? (Aulas 3, 4 e 13)

O projeto mostrou que validar em apenas um ponto não basta. O bug05 (`nome = nome`) estava no construtor — o local ideal para garantir que o objeto nasce em estado válido; sem `this.nome`, o objeto existia mas estava incompleto. O bug04 (`temCreditosSuficientes` invertido) era validação de regra de negócio em método do model — o lugar certo para lógica de domínio que não depende de entrada externa. O bug10 (duração negativa) deveria estar no controller, pois é validação de entrada vinda do cliente (fronteira da aplicação). Campos vindos do JSON chegam por setters chamados pelo Jackson: se `setDuracaoMinutos` não validasse, um setter poderia aceitar valores inválidos antes mesmo do controller agir. A conclusão é: construtor blinda o estado inicial, setters blindam mutações, métodos de negócio blindam regras de domínio, e controllers blindam entrada externa — cada camada tem sua responsabilidade e nenhuma substitui a outra.

### 6. Abstração e interface (Aulas 8 e 9)

`Conteudo` é abstrata porque representa um conceito que não existe sozinho — ninguém aluga um "conteúdo genérico", apenas um Filme, Série ou Documentário. A classe abstrata define a estrutura comum (id, título, categoria, duração) e o método `calcularPrecoAluguel()` como ponto de extensão para cada subclasse. `Promocionavel` é uma interface porque descreve uma capacidade opcional — nem todo conteúdo tem promoção. Filme e Série implementam `Promocionavel`; Documentário não. Se o Documentário passasse a ter promoções, bastaria adicionar `implements Promocionavel` na declaração de `Documentario` e implementar `aplicarPromocao(double preco)` com a lógica de desconto — nenhuma outra classe precisaria ser tocada. `Conteudo.calcularPrecoPromocional()` já usa `instanceof Promocionavel` para verificar e aplicar a promoção automaticamente, então o fluxo de aluguel funcionaria sem alteração. Isso demonstra que a interface segregou bem o comportamento: código que depende de `Promocionavel` é agnóstico ao tipo concreto de conteúdo.

---

## Parte 4 — Espaço livre (opcional)

```
O maior desafio foi identificar o bug03 (overload vs override em Serie), pois o código compilava
sem erros e o comportamento errado (cobrar R$9,90) só ficava claro ao testar o endpoint de preço
ou ao ler a classe comparando com a superclasse. A anotação @Override, se tivesse sido usada,
teria exposto o problema imediatamente em tempo de compilação.
```
