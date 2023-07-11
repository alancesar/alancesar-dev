---
title: Um pouco sobre arquivos e o pacote io do Go
date: 2021-11-13T15:20:34.121Z
description: Trabalhar com arquivos em Golang pode parecer algo trivial, porém é
  comum aparecer algumas dúvidas. Tentarei esclarecer algumas delas
featured: "images/featured.jpg"
---
Trabalhar com arquivos em Golang pode parecer algo trivial, porém é comum aparecer algumas dúvidas. A maneira mais prática de abrir um arquivo é com o comando `os.Open()`:

```go
file, err := os.Open("file.path")
```

Esse comando recebe um caminho e irá abrir o arquivo com permissões padrões, que no caso são somente para leitura.

```go
// Open opens the named file for reading. If successful, methods on
// the returned file can be used for reading; the associated file
// descriptor has mode O_RDONLY.
// If there is an error, it will be of type *PathError.
func Open(name string) (*File, error) {
	return OpenFile(name, O_RDONLY, 0)
}
```

Caso seja necessário realizar alterações no arquivo, precisamos usar o [os.OpenFile()](https://pkg.go.dev/os#OpenFile), que recebe mais dois argumentos: uma *flag* e uma permissão. A permissão segue o [padrão Unix](https://docs.nersc.gov/filesystems/unix-file-permissions/) e essas são as *flags* disponíveis:

```go
const (
	// Exactly one of O_RDONLY, O_WRONLY, or O_RDWR must be specified.
	O_RDONLY int = syscall.O_RDONLY // open the file read-only.
	O_WRONLY int = syscall.O_WRONLY // open the file write-only.
	O_RDWR   int = syscall.O_RDWR   // open the file read-write.
	// The remaining values may be or'ed in to control behavior.
	O_APPEND int = syscall.O_APPEND // append data to the file when writing.
	O_CREATE int = syscall.O_CREAT  // create a new file if none exists.
	O_EXCL   int = syscall.O_EXCL   // used with O_CREATE, file must not exist.
	O_SYNC   int = syscall.O_SYNC   // open for synchronous I/O.
	O_TRUNC  int = syscall.O_TRUNC  // truncate regular writable file when opened.
)
```

Abrir um arquivo não necessariamente significa carregar ele em memória na aplicação. É apenas criado um controle de leitura (e escrita, se for o caso) dentro do sistema operacional e algumas informações básicas, como nome e tamanho, são carregadas. Por isso é possível abrir arquivos maiores que a memória RAM disponível sem maiores problemas. Também é importante destacar que a API `os.File` implementa quatro interfaces do pacote [io](https://pkg.go.dev/io), que são elas:

- [io.Reader](https://pkg.go.dev/io#Reader)
- [io.Writer](https://pkg.go.dev/io#Writer)
- [io.Seeker](https://pkg.go.dev/io#Seeker)
- [io.Closer](https://pkg.go.dev/io#Closer)

## io.Reader

Utilizar essa interface é como assistir uma partida de futebol ao vivo. Não há como "rebobinar" (jovens, me desculpe, não consigo pensar em outra expressão) e voltar ao pontapé inicial; a partida segue conforme o relógio avança. Essa interface possui somente o método `Read()`, que recebe como parâmetro um array de bytes e retorna um inteiro, que é a quantidade de bytes lidos e um `error`, que será `nil`, caso tudo dê certo. O array do parâmetro será usado como um buffer, ou seja, o seu tamanho será a quantidade máxima de bytes lidos.

```go
func TestRead(t *testing.T) {
	reader := strings.NewReader("neste buffer cabe somente 10 caracteres\n")
	buffer := make([]byte, 10)
	numOfBytesRead, err := reader.Read(buffer)
	if err != nil {
		log.Fatalln(err)
	}

	fmt.Printf("bytes lidos: %d\n", numOfBytesRead)
	fmt.Println(string(buffer))
}
```

Saída no console:

```
=== RUN   TestRead
bytes lidos: 10
neste buff
--- PASS: TestRead (0.00s)
PASS

Process finished with the exit code 0
```

Conforme o arquivo é lido, um ponteiro interno do reader avança; então, na próxima vez que o método foi invocado, ele irá continuar a leitura de onde parou e assim sucessivamente até que chegue ao final do arquivo, retornando o error `io.EOF` (end of file). Por conta disso, não é possível ler um `io.Reader` mais de uma vez, porém há formas de se resolver isso que irei detalhar adiante.

```go
func TestRead(t *testing.T) {
	reader := strings.NewReader("neste buffer cabe somente 10 caracteres\n")

	for {
		buffer := make([]byte, 10)
		if _, err := reader.Read(buffer); err == io.EOF {
			fmt.Println("leitura finalizada")
			return
		}

		fmt.Print(string(buffer))
	}
}
```

Saída no console:

```
=== RUN   TestRead
neste buffer cabe somente 10 caracteres
leitura finalizada
--- PASS: TestRead (0.00s)
PASS

Process finished with the exit code 0
```

O método `io.ReadAll()` é um utilitário que lê em memória todo o conteúdo de um `io.Reader`, encapsulando a lógica de concatenar todos os buffers em um só array. É importante lembrar que este método consome memória RAM, portanto deve ser utilizado com cautela em rotinas que lidam com arquivos muito grandes.

## io.Writer

Tudo o que foi dito para o `io.Reader` se aplica para o `io.Writer`, mas, evidentemente, para operações de escrita. Contudo, não existe um método `WriteAll()`, porque é presumido que você tenha disponível em memória todo o conteúdo que deve ser gravado. Porém, o Go disponibiliza o método [io.Copy()](https://pkg.go.dev/io#Copy) que permite enviar dados de um `Reader` para um `Writer`. Também há o método [io.TeeReader()](https://pkg.go.dev/io#TeeReader), inspirado no comando [tee](https://en.wikipedia.org/wiki/Tee_(command)) do Unix, que retorna um `Reader` que escreve em um `Writer` tudo que é lido de outro `Reader` 🤯.

```go
func TestTeeReader(t *testing.T) {
	reader := strings.NewReader("assim funcionavam os antigos aparelhos VCR")
	writer := new(bytes.Buffer)

	// tee vai ler do reader e escrever no writer
	tee := io.TeeReader(reader, writer)
	output, _ := io.ReadAll(tee)
	fmt.Println(string(output))
	fmt.Println(writer.String())
}
```

Saída do console:

```
=== RUN   TestTeeReader
assim funcionavam os antigos aparelhos VCR
assim funcionavam os antigos aparelhos VCR
--- PASS: TestTeeReader (0.00s)
PASS

Process finished with the exit code 0
```

## io.Seeker

Se o `io.Reader` é um futebol ao vivo, o `io.Seeker` é um serviço de *streaming*. O método `Seek` permite navegar pelo conteúdo de um arquivo ou stream, tornando possível iniciar uma leitura ou escrita a partir de determinado ponto. Sendo assim, se for necessário ler algum conteúdo já lido anterior, bastaria apenas retornar o ponteiro à posição zero.

```go
func TestSeek(t *testing.T) {
	reader := strings.NewReader("um reader nem sempre pode ser lido somente uma vez")
	bytes, _ := io.ReadAll(reader)
	fmt.Println(string(bytes))

	_, _ = reader.Seek(0, 0)
	moreBytes, _ := io.ReadAll(reader)
	fmt.Println(string(moreBytes))
}
```

Saída do console:

```go
=== RUN   TestTeeReader
um reader só pode ser lido uma vez
um reader só pode ser lido uma vez
--- PASS: TestTeeReader (0.00s)
PASS

Process finished with the exit code 0
```

## io.Closer

O método `Close()` aciona tudo o que precisa ser feito pelo sistema operacional para que esse recurso seja liberado.

## Variações

Além dessas quatro interfaces mencionadas, o pacote `io` fornece também algumas variações, como o `WriterAt` e `ReaderAt` (ambas implementadas pelo `os.File`) que permite ler ou escrever em uma posição específica, além de combinações como o `ReadSeeker` ou `WriteCloser`, entre outras, que nada mais são do que composições daquelas interfaces básicas.

## Boas práticas

Usar o ponteiro de `os.File` como parâmetro pode acabar não sendo muito vantajoso, já que as interfaces do pacote `io` deixa o código desacoplado da implementação, consequentemente, tornando-o mais fácil de testar. É tentador ler todo o conteúdo de um `Reader` quando se deseja realizar várias atividades em paralelo, como gerar um *hash* e extrair um cabeçalho enquanto faz upload para um outro serviço externo. Porém, isso pode trazer problemas de performance e prejudicar a saúde da sua aplicação, já que, como dito, isso consome a memória RAM.

As APIs do Go fazem um bom trabalho de otimização, como o método [ParseMultipartForm](https://pkg.go.dev/net/http#Request.ParseMultipartForm) do `http.Request` que combina o uso de memória e disco para criar fragmentos do arquivo. Estudar e entender seu comportamento pode trazer dicas valiosas.
