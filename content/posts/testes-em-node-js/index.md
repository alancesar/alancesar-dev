---
title: Testes em Node.js
date: "2017-12-15T12:00:00.0Z"
featured: "images/featured.jpg"
---
Dentre todas as técnicas que podem usadas para melhorar o código que escrevemos, uma que sempre merece destaque são os testes unitários. Neste artigo iremos tratar, através exemplos, como realizar testes unitários em uma aplicação Node.js. Como pré-requisito, você deve possuir um conhecimento básico em Node.js e alguma familiaridade com [ECMAScript 2015 (ES6)](http://www.ecma-international.org/ecma-262/6.0/). Apresentaremos alguns conceitos sobre testes unitários, tais como mocks e asserts.

## Projeto Exemplo
Para rodar nossos testes vamos usar um projeto exemplo chamado **Bookstore**, que pode ser baixado no [GitHub](https://github.com/alancesar/bookstore/). Se trata de um projeto bem simples cuja estrutura é a seguinte:

```text
bookstore
  - services
    - BookService.js // serviço que será testado
  - package.json
```

No diretório `services` ficará nosso serviço a ser testado, chamado aqui de `BookService`, salvo em um arquivo de mesmo nome, `BookService.js`:

```javascript
const rp = require('request-promise');

const API_KEY = 'SPNROBO0';

class BookService {
  static getBookByIsbn(isbn) {
    if (!isbn) {
      throw new Error('ISBN deve ser informado!');
    }

    const options = {
      uri: 'http://isbndb.com/api/v2/json/' + API_KEY + '/book/' + isbn,
      json: true,
    };

    return rp.get(options).then((response) => {
      if (response.error) {
        return {};
      }

      const { title, author_data, publisher_name, language } = response.data[0];

      return {
        title,
        authors: author_data.map(author => author.name).join(';'),
        publisher: publisher_name,
        language,
      };
    });
  }
}

module.exports = BookService;
```

Esse serviço faz uma consulta à API REST do [ISBNdb](http://isbndb.com/), onde dado um código ISBN, é retornado os dados do livro correspondente.

As boas práticas nos dizem para mantermos uma nomenclatura padrão em nossos arquivos de testes algo como `*.test.js` ou `*.spec.js`. Isto facilita ao criarmos os scripts `npm` encarregados de executar nosso conjunto de testes. Também é comum que se tenha diretório `/tests` na aplicação, onde os testes são salvos. Outra abordagem é mantermos os arquivos de testes no mesmo diretório do código que está sendo testado usando a nomenclatura `arquivo-de-código.spec.js`. Neste artigo optamos por esta estratégia e aqui teremos por exemplo: `BookService.js` implicando em `BookService.spec.js`.

```text
bookstore
  - services
  - BookService.js // serviço que será testado
  - BookService.spec.js // especificação do teste - arquivo de teste
- package.json
```

## Ambiente para execução de testes
Para executar nossos testes, precisamos instalar um motor de execução testes JavaScript. No caso usaremos o Mocha, mas como alternativas podemos citar também o [Jest](https://facebook.github.io/jest) e o [Jasmine](https://jasmine.github.io/).
O [Mocha](https://mochajs.org/) é um framework de testes assíncronos em JavaScript que funciona tanto no navegador quanto em aplicações Node.js. Vale ressaltar que o Mocha dispõe de mais ferramentas, além das de testes unitários. Você pode conferir todas aqui.
Além do motor para execução dos testes precisamos de uma ferramenta de asserção para validar os testes, num primeiro momento vamos usar ao [assert](https://nodejs.org/api/assert.html) nativo do Node.js.

Como o Mocha será necessário somente no ambiente de desenvolvimento, informaremos o parâmetros `--save-dev` ao `npm install` para que as dependências não sejam baixadas pelo servidor da aplicação em produção, por exemplo.

```shell
npm install --save-dev mocha
```

Uma vez que tenhamos o Mocha instalado vamos configurar o script npm test responsável por rodar nosso conjunto de testes. No arquivo `package.json`, localize o seguinte trecho:

```json
"scripts": {
  "test": "echo \"Error: no test specified\" && exit 1"
},
```

Vamos configurar o script `test` desta forma:

```json
"scripts": {
  "test": "./node_modules/.bin/mocha './!(node_modules)/**/*.spec.js'"
},
```

Assim quando executarmos o script:

```shell
npm run test
```

O Mocha irá varrer todos os diretórios do projeto, com exceção do `node_modules` (que é onde são armazenadas as dependências do nosso projeto) e executará cada um dos arquivos definimos com a extensão `spec.js`. Ao ser executado, o próprio Mocha é se encarrega de disponibilizar globalmente, no contexto dos testes, todas as dependências necessárias. Desta forma, não temos o trabalho de importar dependências como: `describe` e `it`. Vale lembrar que, caso seja adotado um outro padrão de diretório ou nomenclatura, este script precisa ser readequado.

Tudo pronto? Então já podemos começar a escrever nossos testes. Vamos criar o `BookService.spec.js`.

O comando describe disponibilizado pelo Mocha é usado descrever uma suíte de testes. Ele é uma função que recebe dois parâmetros: o nome da nossa suíte de testes e uma função callback, onde um ou mais testes que serão executados. Já o comando `it` é usado para definir um teste em particular. Ele também recebe dois parâmetros, uma string para descrição e uma função de callback contendo o teste em si. O resultado é algo mais ou menos assim:

```javascript
const assert = require('assert');

describe('BookService.js', () => {
  it('Meu primeiro teste!', () => {
    const a = 1;
    const b = 2;
    const c = a + b;

    assert.equal(c, 3);
  });
});
```

Agora, vamos rodar esse teste. Execute o comando npm test e o resultado no console será:

```shell
npm test

  BookService.js
    ✓ Meu primeiro teste!

  1 passing (6ms)
```

Legal, já é um começo, rodamos nosso teste e temos algum resultado. Mas que tal testarmos algo mais… como posso dizer… útil? Então, vamos escrever um teste o método `getBookByIsbn()`, porém aqui estaremos invocando o método *as is*, ou seja, o método real e portanto ocorrerá uma chamada o serviço de verdade.

```javascript
const assert = require('assert');
const BookService = require('./BookService');

describe('BookService.js', () => {
  ...
  it('Requisição válida e título do livro', () => {
    // Given
    const ISBN = '0345391802';

    // When
    const result = BookService.getBookByIsbn(ISBN);

    // Then
    return result.then((book) => {
      assert.equal(book.title, 'The hitchhiker\'s guide to the galaxy');
    });
  });
});
```

Assim, o teste irá executar uma chamada no nosso serviço e verificar o título do livro. Então, execute mais uma vez o comando `npm test` e teremos o resultado:

```shell
npm test

  BookService.js
    ✓ Meu primeiro teste!
    ✓ Requisição válida e título do livro (292ms)

  2 passing (310ms)
```

Você lembra que dissemos que o Mocha é um framework de testes assíncronos? Então, precisamos retornar uma promise no teste, já que nosso serviço é assíncrono.

```javascript
return result.then(book => {
  assert.equal(book.title, 'The hitchhiker\'s guide to the galaxy');
});
```

Ou seja, o Mocha executa todos os testes, mas não fica aguardando as respostas de promises. Então, devemos informar quando o teste será concluído. Para isso, podemos retornar um promise como feito acima, ou utilizamos a função callback `done`. Que é um argumento da função it e quando invocado sinaliza ao Mocha que o teste está finalizado, segue um exemplo:

```javascript
const assert = require('assert');
const BookService = require('./BookService');

describe('BookService.js', () => {
  ...
  it('Requisição válida e título do livro com done()', (done) => {
    //GIVEN
    const ISBN = '0345391802';
    const bookService = new BookService();

    //WHEN
    const result = bookService.getBookByIsbn(ISBN);

    //THEN
    result.then(book => {
      assert.equal(book.title, 'The hitchhiker\'s guide to the galaxy');
      done(); // meu teste terminou;
    });
  });
});
```

Rodando o teste novamente teremos um resultado equivalente ao anterior:

```shell
npm test

  BookService.js
    ✓ Meu primeiro teste!
    ✓ Requisição válida e título do livro (408ms)
    ✓ Requisição válida e título do livro com done() (273ms)
```

## Bibliotecas assertivas

Um teste é basicamente checar se a saída do nosso método é aquilo que esperávamos. Embora a biblioteca nativa assert atenda suficientemente bem esse papel, podemos fazer uso de ferramentas mais poderosas como é o caso do Chai. Para instalar basta executar o comando:

```shell
npm install --save-dev chai
```

O Chai possui o componente expect. Com ele podemos fazer diversas comparações e análises.  Seus métodos e atributos possuem sintaxes simples e semânticas, suporte a metodologia [BDD/TDD](http://toolsqa.com/cucumber/behavior-driven-development/) e utiliza o padrão de [interfaces fluentes](https://martinfowler.com/bliki/FluentInterface.html), tornando extremamente fácil ler e escrever códigos de teste.

Vamos escrever um teste que verifica se todos os atributos do nosso objeto estão corretos:

```javascript
const assert = require('assert');
const { expect } = require('chai');
const BookService = require('./BookService');

describe('BookService.js', () => {
  ...
  it('Requisição válida e retorno com expect', () => {
    // Given
    const ISBN = '0345391802';
    const bookService = new BookService();
    const expected = {
      authors: 'Adams, Douglas',
      language: 'eng',
      publisher: 'Harmony Books',
      title: 'The hitchhiker\'s guide to the galaxy',
    };

    // When
    const result = bookService.getBookByIsbn(ISBN);

    // Then
    return result.then((book) => {
      expect(book).to.be.deep.equal(expected); // Usando ‘expect’ do Chai
    });
  });
});
```

O método de comparação descreve de forma clara o que está sendo realizado. É importante notar que estamos utilizando o método `deep.equal` a comparação é feita pelo valor de cada atributo e não pela referência em memória do objeto.

O Chai possui diversos outros métodos assertivos para as mais diversas situações. Você pode consulta-los na [documentação oficial](http://chaijs.com/api/bdd) sempre que um deep.equal não for o suficiente.

Pronto, nosso teste está funcionando. Mas e aquela mensagem de milissegundos em vermelho? Bom, vamos resolver isso.

## Mockagem

A mensagem dos milissegundos se deve ao fato de estarmos fazendo requisições HTTP à API do ISBNdb. Em testes unitários, isso é uma péssima prática – até porque são unitários, não de integração. Já que estamos escrevendo testes unitários, não faz sentido realizar uma chamada real a API. Precisamos simplesmente de um resultado válido para testar a lógica do nosso método.

Para a nossa alegria, temos o Sinon, que é uma biblioteca específica para isso  e pode ser instalado como segue:

```shell
npm i --save-dev sinon
```

O Sinon faz uso da técnica [Monkey Path](http://me.dt.in.th/page/JavaScript-override), isto é, substitui os métodos originais dos componentes por métodos que simulem as iterações e retornos do método real.

Vamos observar nossa classe `BookService`. É feita uma requisição do tipo `GET` através do `request-promise`, que nos retorna o resultado da busca. Então, é esse método que devemos mockar.

O Sinon tem a capacidade de alterar componentes mesmo que eles não sejam invocados diretamente nos testes, embora necessitem serem importados para funcionarem adequadamente.

Nossa infraestrutura ficará assim:

```javascript
const chai = require('chai');
const {expect} = require('chai');
const sinon = require('sinon');
const rp = require('request-promise');
const BookService = require('../services/BookService');

describe('BookService.js', () => {
  ...
});
```

As novidades até aqui são o import do Sinon e do request-promise. O Sinon possui o método `stub()`. Com ele, podemos alterar o comportamento de um método. Vamos fazer com que o request-promise usado no método `getBookByIsbn()` na classe `BookService` retorne um valor de acordo com a necessidade do nosso teste.

```javascript
const assert = require('assert');
const { expect } = require('chai');
const sinon = require('sinon');
const rp = require('request-promise');
const BookService = require('./BookService');

describe('BookService.js', () => {
  ...
  it('Válida e retorno com expect e mock', () => {
    // Given
    const ISBN = '0345391802';
    const expected = {
      authors: 'Adams, Douglas',
      language: 'eng',
      publisher: 'Harmony Books',
      title: 'The hitchhiker\'s guide to the galaxy',
    };

    // Preparando o Stub/Mock
    const stubRpGet = sinon.stub(rp, 'get');
    stubRpGet.resolves({
      data: [{
        author_data: [{ name: 'Adams, Douglas' }],
        language: 'eng',
        publisher_name: 'Harmony Books',
        title: 'The hitchhiker\'s guide to the galaxy',
      }],
    });

    // When
    const result = BookService.getBookByIsbn(ISBN);

    // Then
    return result.then((book) => {
      expect(book).to.be.deep.equal(expected);
      stubRpGet.restore(); // Restaurando o método original
    });
  });
});
```

O `stub()` recebe dois argumentos. O primeiro é a função, classe ou objeto que desejamos mockar. O segundo é uma string com o nome do método. Ele retorna um objeto contendo o método modificado do nosso alvo. Como o `get()` da `request-promise` retorna uma promise, podemos simplesmente invocar o método `resolves()` e passar como argumento o que deverá ser devolvido pela promise do `get()`, nesse caso, o JSON com os dados do livro.

Caso o método `get()` não fosse uma promise, poderíamos usar o método `returns()`, que faz um retorno de uma chamada síncrona comum.

Se executarmos novamente o teste, veremos que o tempo que execução de nosso testes reduziu consideravelmente e não está mais vermelho.

```shell
npm test

  BookService.js
    ✓ Meu primeiro teste!
    ✓ Requisição válida e título do livro (259ms)
    ✓ Requisição válida e título do livro com done() (236ms)
    ✓ Válida e retorno com expect (245ms)
    ✓ Válida e retorno com expect e mock

  5 passing (770ms)
```

Um detalhe importante é que o métodos não-estáticos, isto é, que dependem da instância de uma classe, precisam ser declarados através do prototype, da seguinte forma:

```javascript
const stubRpGet = sinon.stub(MinhaClasse.proptotype, 'metodoNaoEstatico');
```

Mesmo assim, este teste ainda não garante que nosso serviço funcionará da maneira correta. Algum desenvolvedor mais desavisado pode achar a URL de requisição feia demais e remover o parâmetro `API_KEY` que está dentro da classe do nosso serviço.

```javascript
const options = {
  uri: 'http://isbndb.com/api/v2/json/book/' + isbn,
  json: true,
};
```

O teste continuaria passando, mas teríamos problemas quando este serviço fosse utilizado em produção. O pessoal do Sinon também pensou nisso. Nosso objeto não foi modificado somente em seu comportamento, mas recebeu diversas funcionalidades que podem ser acessadas, nesse exemplo, através de `stubRpGet`. Uma destas funcionalidades é verificar os parâmetros que nosso método mockado recebeu.

Dessa forma, se imprimirmos no console `stubRpGet.args`, veremos nossa URL problemática. Poderíamos continuar fazendo a asserção utilizando o `expect()` e comparando com os argumentos esperados, mas o Sinon nos fornece uma infraestrutura para diversas comparações:

```javascript
return result.then((book) => {
  expect(book).to.be.deep.equal(expected); // Antes
  book.should.to.be.deep.equal(expected);  // Depois

  // Podemos melhorar!
  const API_KEY = 'SPNROBO0';
  stubRpGet.should.be.calledWith({
    uri: 'http://isbndb.com/api/v2/json/' + API_KEY + '/book/' + ISBN,
    json: true,
  });

  // Restaurando o método original
  stubRpGet.restore();
});
```
Dessa forma, caso algo fosse alterado na URL da classe BookService nosso teste quebraria, nos mostrando que a requisição está incorreta e teríamos a possibilidade de corrigir o problema.

Caso você tenha alterado a variável options do BookService para testar o erro, lembre-se de corrigi-la.

## Should I stay or should I go?

Vamos ativar um recurso interessante do Chai:

```javascript
const chai = require('chai');
chai.should();
...

describe('BookService.js', () => {
  ...
}
```

O Should remove a necessidade do método expect e deixa nossos testes ainda mais semânticos. Veja como ele ficará:

```javascript
return result.then(book => {
  expect(book).to.be.deep.equal(expected); // Antes
  book.should.to.be.deep.equal(expected);  // Depois
  ...
});
```

O `should` pode receber os mesmos [chains que já vimos no expect](http://chaijs.com/api/bdd/).

Agora, vamos dar alguns superpoderes ao nosso mock. Para isso, faremos uso de uma outra biblioteca interessante:

```shell
npm i --save-dev sinon-chai
```

O Sinon-Chai acopla a assertividade do Chai ao Sinon. Devemos fazer uma configuração simples antes:

```javascript
const chai = require('chai');
const sinonChai = require('sinon-chai');
...

chai.use(sinonChai);
```

Então, poderemos fazer a assertividade do Sinon da mesma forma que usando should do Chai:

```javascript
  stubRpGet.should.be.calledWith({
    uri: 'http://isbndb.com/api/v2/json/' + API_KEY + '/book/' + ISBN,
    json: true
  });
```

A cobertura de testes de nossa aplicação está quase completa. Falta agora cobrir em caso de falha. Antes disso, um detalhe importante. Quando um mock é aplicado a um método, esse objeto mantém-se alterado durante a execução de todos os testes. Existem cenários em que pode ser desejável realizar esse mock de maneiras diferentes em cada teste, então antes de fazermos isso, precisamos invocar o método `restore()`, para que este método volte ao seu estado original. Mas caso tenhamos vários testes, pode-se tornar trabalhoso demais fazer isto em cada um deles. Então podemos ter auxílio de hooks. Temos o beforeEach e afterEach, que são executados antes e depois de cada teste, respectivamente. Esses hooks devem ser declarados dentro da instrução describe. Então, nosso código ficará assim:

```javascript
describe('BookService.js', () => {
  afterEach(() => {
    if (rp.get.restore) {
      rp.get.restore();
    }
  });
...
});
```

Temos também a função `skip`, que permite que um teste seja ignorado:

```javascript
it.skip('Teste a ser ignorado', () => {
  ...
});
```

Há também a função `only`, que executa somente os teste marcados:

```javascript
it.only('Somente este teste será executado', () => {
  ...
});
```

As instruções `skip` e `only` também podem ser aplicadas ao describe.

```javascript
describe.only('BookService.js', () => {
  ...
});
```

Agora, vamos ao código para testar uma requisição inválida, ou seja, sem informar o ISBN:

```javascript
it('Requisição inválida', () => {
  BookService.getBookByIsbn().should.to.be.throws('ISBN deve ser informado!');
});
```

Observe que o Chai possui a instrução throws, que verifica se a execução devolveu uma exceção. Podemos especificar ainda qual a mensagem ou tipo de erro deve ser retornado, para garantir que aquela exceção é exatamente a esperada. Porém, ao observar o console vemos:

```shell
npm test

  1) ConsultaLivroService.js Requisição inválida:
     Error: ISBN não encontrado!
```

O erro foi lançado, porém, não foi validado pelo Chai. O que acontece é a exceção do nosso código está no mesmo nível que o should. Quando este throws é lançado, o should não é invocado, mas sim uma exceção do próprio it. Para resolver isto, precisamos isolar a execução deste código dentro do próprio Chai:

```javascript
it('Requisição inválida', () => {
  chai.expect(() => {
    BookService.getBookByIsbn();
  }).to.be.throws('ISBN deve ser informado!');
});
```

Dessa forma, temos total cobertura de testes do nosso serviço. O conteúdo mostrado neste tutorial deve cobrir a maior parte dos casos de testes, mas não exite em consultar a documentação destes frameworks caso precise de informações mais detalhadas!

Também colaborou com esse post [Willian Batista](https://github.com/wfuertes).
