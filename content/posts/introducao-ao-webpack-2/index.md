---
title: Introdução ao Webpack 2
date: "2017-07-10T12:00:00.0Z"
---

O Webpack é um empacotador de código-fonte para projetos web. Diferente de outros similares, como o [browserify](http://browserify.org/), sua principal proposta é focar na modularização da aplicação. A justificativa até que é simples: para grandes projetos web, não é eficiente colocarmos todo código em um único arquivo, especialmente se alguns blocos de código forem necessários somente em algumas determinadas circunstâncias.

![Diagrama do Webpack](webpack.png)

Dentre as principais características do Webpack temos:
- code splitting;
- loaders;
- clever parsing;
- plugin system.

### Instalação
Sua instalação é relativamente simples. O Webpack tem como dependência o NodeJS e pode ser instalado tanto globalmente como  dependência de desenvolvimento para seu projeto, que será o modo utilizado neste exemplo. No terminal, com o NodeJS e NPM instalados, execute os seguintes comando:

```shell
npm init -y
npm install --save-dev webpack
```

### Configuração
Considere um projeto com a seguinte estrutura:

```text
.
├── src
│   ├── css
│       └── main.css
│   └── Mensagem.js
│   └── main.js
├── index.html
├── package.json
└── webpack.config.js
```

Por padrão, o Webpack carrega suas configurações do arquivo `webpack.config.js`, que deve estar localizado na raiz do projeto. Configure-o da seguinte forma:

```javascript
const webpack = require('webpack');
const path = require('path');

module.exports = {
   entry: {
       main: './src/main.js'
   },
   output: {
       path: path.resolve(__dirname, 'dist'),
       filename: 'bundle.js'
   }
}
```

O atributo `entry` define os pontos de entrada da nossa aplicação. Neste exemplo, definimos um ponto de entrada, chamado `app`, que referencia o arquivo `src/main.js`. É possível criar vários pontos de entrada e deixar nossa aplicação ainda mais modularizada, mas neste exemplo será utilizado apenas um, para simplificar as coisas.
Já o atributo `output` é definido os pontos de saída. São nessas saídas que o conteúdo será empacotado.
Para dar sequência ao nosso projeto, vamos criar um script simples em `src/main.js`:

```javascript
function bemVindo() {
   alert('Olá mundo');
}
```

Agora a magia irá acontecer. No terminal, execute o comando:

```shell
./node_modules/.bin/webpack
```

Esse comando é responsável por empacotar todo o projeto, com base nas instruções que foram informadas nas configurações. Perceba que o Webpack criou o diretório `dist`, contendo nosso arquivo `bundle.js`:
{% assign image = page.images[2] %}
{% include image.html image=image %}

### Módulos
Uma das grandes vantagens do Webpack é a possibilidade de trabalhar com módulos. Módulos dão funções extras ao Webpack, deixando a ferramenta ainda mais poderosa. Um módulo famoso e muito utilizado é o Babel. Ele é um transpiler, ou seja, ele converte determinadas anotações e tags para um script nativo do JavaScript, que pode ser lido em qualquer navegador. Podemos escrever, por exemplo, nosso código em EcmaScript 2015 (ES6) que o Babel gerará um arquivo que pode ser lido até em navegadores que ainda não tem esse recurso disponível. Iremos incluí-lo em nosso Webpack, mas antes, precisamos instalar algumas dependências:

```shell
npm install --save-dev babel-cli babel-core babel-loader babel-preset-es2015
```

Agora nosso arquivo de configuração do Webpack está pronto para receber as novas instruções:

```javascript
module.exports = {
   ...
   module: {
       rules: [
           {
               test: /\.js$/,
               exclude: /node_modules/,
               use: [
                   {
                       loader: 'babel-loader',
                       options: {
                           presets: ['es2015']
                       }
                   }
               ]
           }
       ]
   }
}
```

A novidade aqui é o atributo `module`. Nela, definimos as regras, que, neste caso, só possui uma: verificar, através de uma expressão regular, todos os arquivos com extensão `.js`, com exceção do diretório `node_modules` (onde ficam os arquivos de módulos do NodeJS), e aplicar o Babel Loader. O Babel Loader possui alguns presets, ou seja, configurações pré definidas, que podem ser utilizadas conforme a necessidade. Neste caso, utilizamos somente o ES2015.
Para testar se está tudo funcionando, podemos alterar o arquivo `src/main.js` e incluir alguma funcionalidade do ES2015, como o suporte a classes:

``` javascript
class Mensagem {
   bemVindo() {
       document.write('<h1>Olá mundo!</h1>');
   }
}

let mensagem = new Mensagem();
mensagem.bemVindo();
```

Se executarmos novamente `./node_modules/.bin/webpack` e observamos o arquivo `bundle.js`, podemos perceber que o Babel, através do Webpack, é um pouco diferente daquele que escrevemos, mas, ao ser executado no navegador, será interpretado da mesma maneira.

### Server
Durante o desenvolvimento, será constante a necessidade de testar nossa aplicação. Pensando nisso, foi desenvolvido o Webpack Dev Server, que nada mais é que o próprio Webapck com um pequeno servidor HTTP embutido.
Sua instalação também pode ser feita via NPM:

```shell
npm install --save-dev webpack-dev-server
```

Antes de utilizá-lo, na raiz de nosso projeto, vamos criar o `index.html`:

```html
<!DOCTYPE html>
<html>
 <head>
   <title>Tutorial webpack 2</title>
 </head>
 <body>
   <script src="bundle.js"></script>
 </body>
</html>
```

Então, execute o comando

```shell
./node_modules/.bin/webpack-dev-server
```

Este comando irá subir servidor na porta `8080`. Diferentemente de um servidor HTTP tradicional, o Webpack Dev Server irá carregar nossos bundles em memória, então não se assuste se o diretório `dist` não for alterado, ou até mesmo não for criado. Outro recurso interessante é o hot reload: os bundles são gerados e o site é recarregado automaticamente, sempre que algum arquivo for alterado.

### Import
Como vimos, um dos trunfos do Webpack é a capacidade de empacotar os arquivos de nosso projeto. Até então, nosso projeto só possui um arquivo, o que não tirava vantagem desse recurso. Então, vamos mover nossa classe Mensagem para um novo arquivo em `/src/Mensagem.js`:

```javascript
class Mensagem {
   bemVindo() {
       document.write('<h1>Olá mundo!</h1>');
   }
}

export default Mensagem;
```

E faremos a referência a ele em nosso `src/main.js` através da instrução `import`:

```javascript
import Mensagem from './Mensagem';

let mensagem = new Mensagem();
mensagem.bemVindo();
```

### CSS
Com o webpack também é possível a inclusão de estilos CSS ao bundle final. Para que isto seja possível, é necessário a instalação de dois módulos do Webpack: o `style-loader` e o `css-loader`, que, em conjunto, interpretarão o código CSS e o injetará no projeto. Execute o comando abaixo no terminal:

```shell
npm install --save-dev style-loader css-loader
```

E inclua as novas regras nos módulos:

```javascript
...
module.exports = {
   ...
   module: {
       rules: [
           ...
           {
               test: /\.css$/,
               use: ['style-loader', 'css-loader'],
           },
       ]
   }
}
```

Não tem segredo: uma expressão regular irá buscar todos os arquivos `.css` do projeto. Mas aqui, um ponto requer atenção: os módulos são carregados da direita para esquerda, ou seja, primeiro o `css-loader` e depois o `style-loader`. Para que o código compile com sucesso, neste caso, essa ordem deve ser seguida, por questão de dependências entre os módulos.

Criaremos um estilo simples em `src/css/main.css`:

```css
body {
   background-color: #eee;
   color: #555;
}

h1 {
   font-family: Arial, Helvetica, sans-serif;
}
```

E importe esse código no `src/main.js`.

```javascript
import Mensagem from './Mensagem';
import './css/main.css';

let mensagem = new Mensagem();
mensagem.bemVindo();
```

## Plugins
Em algumas circunstâncias, pode ser interessante que os estilos CSS de nosso projeto estejam em um arquivo separado. Isso pode ser feito por meio de plugins.
Plugins, assim como módulos, acrescentam funções extras ao Webpack. Eles são menos orgânicos que os módulos, portanto, precisam de algumas configurações extras.
O plugin Extract Text nos permite extrair códigos para um arquivo separado do bundle. Primeiramente, instale-o pelo terminal com o comando:

```shell
npm install --save-dev extract-text-webpack-plugin
```

Então devemos incluir novas instruções em nosso arquivo de configuração do Webpack:

```javascript
...
const ExtractTextWebpackPlugin = require('extract-text-webpack-plugin');
const extractPlugin = new ExtractTextWebpackPlugin({
   filename: 'bundle.css'
});

module.exports = {
   ...
   module: {
       rules: [
           ...
           {
               test: /.css$/,
               use: extractPlugin.extract({
                   use: ['css-loader']
               })
           }
       ]
   },
   plugins: [
       extractPlugin
   ]
}
```

O que fizemos foi:
- Instanciar o objeto `extractPlugin`, passando como argumento uma configuração, indicando que nosso arquivo irá se chamar `bundle.css`;
- A regra para arquivos `.css` foi substituída. Em seu lugar ficou uma referência que passa essa responsabilidade para nosso plugin. Então, indicamos qual módulo será o middleware dessa conversão. Note que não é mais preciso o `style-loader`. Ele é necessário somente se o código CSS ficar dentro do bundle JavaScript gerado.
- Por fim, declaramos ao pacote quais plugins serão necessários (somente o `webpack-extract-text-plugin`, neste caso).
Agora é necessário também incluir a referência ao arquivo .css gerado no `index.html`.

```html
<!DOCTYPE html>
<html>
 <head>
   <title>Tutorial webpack 2</title>
   <link rel="stylesheet" href="bundle.css">
 </head>
 <body>

   <script src="bundle.js"></script>
 </body>
</html>
```

O que mostramos aqui foi um projeto simples, mas que engloba vários recursos do Webpack que provavelmente podem ser utilizados em qualquer projeto web. Você pode encontrar o código fonte utilizado [aqui](https://github.com/alancesar/tutorial-webpack-2). O Webpack pode ir muito além disso, como empacotamento de imagens, SASS, entre outros. Mas isso é assunto para outro post ;).
