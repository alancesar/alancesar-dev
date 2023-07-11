---
title: Trabalhando com Promises no JavaScript
date: "2017-01-14T12:00:00.0Z"
featured: "images/featured.jpg"
---
Uma das grandes novidades que vieram no *ES6* foram as `Promises`. Mas afinal, o que são? De onde vem?
Do que se alimentam?

### Promessa é dívida
Promise vem do inglês, que significa promessa. E é mais ou menos isso: uma tarefa que é executada de forma
assíncrona com a promessa de devolver alguma resposta, assim que concluída. Não é novidade fazer este tipo de
requisições no JavaScript. Com a ausência de uma API específica, utilizava-se alguns workarounds para que este
recurso funcionasse corretamente.

```javascript
function getData(url, callback) {
  var xhr = new XMLHttpRequest();

  xhr.onload = function() {
    if (xhr.readyState === XMLHttpRequest.DONE) {
      if (xhr.status === 200) {
        callback(xhr.responseText);
      }
    }
  };

  xhr.open('GET', url, true);
  xhr.send();
}
```

Por exemplo, dada esta função acima, eu consigo fazer uma chamada em um web service e, assim que obtiver a resposta,
executo alguma ação com ela, como, por exemplo, exibir no console. Para isso, basta eu passar uma função como
parâmetro, que é o que chamamos de `callback`.

```javascript
getData('https://pointer.alancesar.org/v1/place/-23.525889,-46.678908/pt', function(data) {
  console.log(data)
});
```

Até aí tudo bem. Mas imagine que esta minha requisição falhe. Minha aplicação nunca irá saber. Podemos adaptar o
código para lançar um erro, caso as coisas não aconteçam como o esperado.

```javascript
function getData(url, callback, error) {
  var xhr = new XMLHttpRequest();

  xhr.onload = function() {
    if (xhr.readyState === XMLHttpRequest.DONE) {
      if (xhr.status === 200) {
        callback(xhr.responseText);
      } else {
        error('Erro na requisição!');
      }
    }
  };

  xhr.open('GET', url, true);
  xhr.send();
}
```

Agora, além de uma função `callback`, eu passo como parâmetro outra função para ser executada em caso de falha.

```javascript
getData('https://pointer.alancesar.org/v1/place/-23.525889,-46.678908/pt', function(data) {
  console.log(data)
}, function(msg) {
  console.error(msg)
});
```

Ficou mais complexo? Mas ainda assim funciona, né? Agora, se invés de um `GET`, eu tiver um método `POST`?

```javascript
function postData(url, data, callback, error) {
  var xhr = new XMLHttpRequest();

  xhr.onload = function() {
    if (xhr.readyState === XMLHttpRequest.DONE) {
      if (xhr.status === 200) {
        callback(xhr.responseText);
      } else {
        error('Erro na requisição!');
      }
    }
  };

  xhr.open('POST', url, true);
  xhr.send(JSON.stringify(data));
}
```

E o nosso cliente, que fará o uso deste método, deseja apenas fazer o `POST` e nada mais, pouco se importando se
dará certo ou não?

```javascript
postData('https://www.google.com.br', { 'busca': 'JavaScript' });
```

Olha o que vai acontecer:

```shell
Uncaught TypeError: error is not a function at XMLHttpRequest.xhr.onload (<anonymous>:9:9)
```

O interpretador não encontrou o parâmetro `error`. Mas não mesmo, nosso cliente não o informou. Consegue imaginar
alguma solução?

### O inferno dos callbacks

Para resolver este pequeno problema, sem ter que implementar métodos que nunca retornarão nada, pode ser
acrescentada algumas verificação, tanto no `callback` quanto no `error`.

```javascript
function postData(url, data, callback, error) {
  var xhr = new XMLHttpRequest();

  xhr.onload = function() {
    if (xhr.readyState === XMLHttpRequest.DONE) {
      if (xhr.status === 200) {
        if (callback) {
          callback(xhr.responseText);
        }
      } else {
        if (error) {
          error('Erro na requisição!');
        }
      }
    }
  };

  xhr.open('POST', url, true);
  xhr.send(JSON.stringify(data));
}
```

Dessa forma, caso o parâmetro não seja informado, o método apenas o ignora e vida que segue. Mas começou a ficar
feio.
Agora, pra piorar, imagine a necessidade de executar um `postData()` com os dados obtidos no `getData()`?

```javascript
getData('https://pointer.alancesar.org/v1/place/-23.525889,-46.678908/pt', function(data) {
  postData('https://www.google.com.br', { 'busca': data.result }, function(data) {
    console.log(data)
  }, function(msg) {
    console.error(msg)
  });
}, function(msg) {
  console.error(error)
});
```

Chamamos essa aberração de inferno de callbacks e você deve imaginar porquê. Para resolver este empecilho, foi
introduzido no [ECMAScript 2015](http://www.ecma-international.org/ecma-262/6.0/) as
[Promises](https://developer.mozilla.org/pt-BR/docs/Web/JavaScript/Reference/Global_Objects/Promise). Elas fazem
basicamente o mesmo serviço sujo, só que de uma forma bem mais elegante.

```javascript
function getData(url) {
  return new Promise(function(resolve, reject) {
    var xhr = new XMLHttpRequest();

    xhr.onload = function() {
      if (xhr.readyState === XMLHttpRequest.DONE) {
        if (xhr.status === 200) {
          resolve(xhr.responseText);
        } else {
          reject('Erro na requisição!');
        }
      }
    };

    xhr.open('GET', url, true);
    xhr.send();
  });
}
```

A princípio, parece apenas uma mudança na sintaxe que só deixa o código mais complexo ainda. Mas ao executar as
chamadas, começamos a perceber suas vantagens.

```javascript
getData('https://pointer.alancesar.org/v1/place/-23.525889,-46.678908/pt').then(function(data) {
  console.log(data);
});
```

O método `then()` é executado no sucesso da requisição. É o que foi informado no `resolve()`, quando
implementamos o método. Para tratarmos exceções, temos o `catch()`, que é o que foi passado no método
`reject()`.

```javascript
getData('https://pointer.alancesar.org/v1/place/-23.525889,-46.678908/pt').then(function(data) {
  console.log(data);
}).catch(function(msg) {
  console.error(msg);
});
```

Note que, por retornar um objeto `Promise`, eu posso armazenar tudo isso em uma variável.

```javascript
var promise = getData('https://pointer.alancesar.org/v1/place/-23.525889,-46.678908/pt');

// Algum código que não depende da resposta da requisição
// ...

promise.then(function(data) {
  console.log(data);
});
```

E, de brinde, podemos deixar o código mais enxuto fazendo uso das [Arrow Functions](https://developer.mozilla.org/pt-BR/docs/Web/JavaScript/Reference/Functions/Arrow_functions).

```javascript
function getData(url) {
  return new Promise((resolve, reject) => {
    var xhr = new XMLHttpRequest();

    xhr.onload = function() {
      if (xhr.readyState === XMLHttpRequest.DONE) {
        if (xhr.status === 200) {
          resolve(xhr.responseText);
        } else {
          reject('Erro na requisição!');
        }
      }
    };

    xhr.open('GET', url, true);
    xhr.send();
  });
}

getData('https://pointer.alancesar.org/v1/place/-23.525889,-46.678908/pt')
  .then(data => console.log(data))
  .catch(msg => console.error(msg));
});
```
