---
layout: content
title: Plugins
prev: Metadata
next: Shells
link_prev: /pt/metadados.html
link_next: /pt/shells.html
---

## Protocolo

Plugins usam JSON-RPC sobre stdin/stdout (de maneira parecida aos plugins do VSCode). O protocolo é dividido em dois estágios.

O primeiro estágio do protocolo lida com a descoberta inicial do plugin. Um plugin é iniciado e então solicitado para responder com a sua configuração. Muito semelhante aos comandos, plugins possuem uma assinatura que usam para responder ao Nu. Assim que o Nu possuir essa assinatura, ele saberá invocar o plugin futuramente.

O segundo estágio é a realização do verdadeiro trabalho. Aqui é enviado tanto uma stream de dados para o plugin agir em cada elemento como um filtro, ou então todos elementos de uma vez para o plugin agir em um processamento final como uma saída.

## Descoberta

Nu descobre plugins ao checar todos os diretórios disponíveis no PATH atual.
Em cada diretório, Nu busca por arquivos executáveis que combinam com o padrão  `nu_plugin_*`, onde `*` é no mínimo um caracter alfanumérico.
No Windows, isso é um padrão similar à `nu_plugin_*.exe` ou `nu_plugin_*.bat`.

Assim que um arquivo que combine com o padrão for descoberto, Nu vai invocar o arquivo e passar ao primeiro comando JSON-RPC: config.
Config responde com a assinatura do plugin, que é idêntico a assinatura usada por comandos.

Nu continua essa busca até ter percorrido todos os diretórios no caminho.

Após ter percorrido o caminho, dois outros diretórios serão verificados o diretório alvo/debug e o diretório alvo/release. Um ou outro diretório vai ser buscado, dependendo seo  Nu foi compilado no módo de depuração ou de release, respectivamente. Isso permite testar rapidamente os plugins durante o desenvolvimento.

## Criando um plugin (em Rust)

Nessa seção, vamos mostrar como criar um plugin para o Nu usando Rust.

Vamos criar nosso projeto. Para esse exemplo, vamos criar um simples comando `len` que retorna o tamanho da string que recebe.

Primeiramente, vamos criar nosso plugin:

```
> cargo new nu_plugin_len
> cd nu_plugin_len
```

Então, vamos adicionar `nu` na lista de dependências do diretório Cargo.toml. No final do novo arquivo Cargo.toml, adicione essa nova dependência para o crate `nu`:

```
[dependencies]
nu-plugin = "~0"
nu-protocol = "~0"
nu-source = "~0"
nu-errors = "~0"
```

Com isso, podemos abrir src/main.rs e criar nosso plugin.

```rust
use nu_errors::ShellError;
use nu_plugin::{serve_plugin, Plugin};
use nu_protocol::{
    CallInfo, Primitive, ReturnSuccess, ReturnValue, Signature, UntaggedValue, Value,
};

struct Len;

impl Len {
    fn new() -> Len {
        Len
    }

    fn len(&mut self, value: Value) -> Result<Value, ShellError> {
        match &value.value {
            UntaggedValue::Primitive(Primitive::String(s)) => Ok(Value {
                value: UntaggedValue::int(s.len() as i64),
                tag: value.tag,
            }),
            _ => Err(ShellError::labeled_error(
                "Unrecorgnized type in stream",
                "'len' given non-string info by this",
                value.tag.span,
            )),
        }
    }
}

impl Plugin for Len {
    fn config(&mut self) -> Result<Signature, ShellError> {
        Ok(Signature::build("len").desc("My custom len plugin").filter())
    }

    fn begin_filter(&mut self, _: CallInfo) -> Result<Vec<ReturnValue>, ShellError> {
        Ok(vec![])
    }

    fn filter(&mut self, input: Value) -> Result<Vec<ReturnValue>, ShellError> {
        Ok(vec![ReturnSuccess::value(self.len(input)?)])
    }
}

fn main() {
    serve_plugin(&mut Len::new());
}
```

Existe bastante código até aqui, então vamos verificar cada trecho separadamente.

Primeiramente, vamos olhar o main:

```rust
fn main() {
    serve_plugin(&mut Len::new());
}
```

No main, simplesmente chamamos uma única função `serve_plugin`. Isso vai fazer o trabalho de chamar o plugin, lidando com a serialização/desserialização do JSON, e enviando valores e erros de volta para o Nu. Para iniciá-lo, passamos  algo que implementa a trait `Plugin`.

Em seguida, acima do main, está a implementação da trait `Plugin` para o nosso plugin em particular. Aqui, vamos implementar a trait Plugin para o nosso tipo, Len, que veremos em breve. Vamos ver como implementamos essa trait:

```rust
impl Plugin for Len {
    fn config(&mut self) -> Result<Signature, ShellError> {
        Ok(Signature::build("len").desc("My custom len plugin").filter())
    }

    fn begin_filter(&mut self, _: CallInfo) -> Result<Vec<ReturnValue>, ShellError> {
        Ok(vec![])
    }

    fn filter(&mut self, input: Value) -> Result<Vec<ReturnValue>, ShellError> {
        Ok(vec![ReturnSuccess::value(self.len(input)?)])
    }
}
```

As duas partes mais importantes dessa implementação  são a parte de `config`, que é executada pelo Nu quando se inicia pela primeira vez. Isso informa ao Nu as informações básicas sobre o plugin: nome, parâmetros recebidos, descrição e qual o tipo do plugin.
Aqui, informamos ao Nu que o nome é "len", damos uma básica descrição de `ajuda` para mostrar e que somos um plugin de filtro (ao invés de um plugin de saída).

Em seguida, na implementação do `filter`, descrevemos como as informações são processadas com o fluxo de dados neste plugin. Aqui, recebemos um valor (um `Value`) de cada vez.
Também retornamos ou um Vec de valores ou um erro.
Retornar um vec ao invés de um único valor nos permite remover valores, ou adicionar outros, além de trabalhar com o único valor recebido.
  
Já que o `begin_filter` não faz nada, podemos remove-lo. Isso simplificaria o código acima:

```rust
impl Plugin for Len {
    fn config(&mut self) -> Result<Signature, ShellError> {
        Ok(Signature::build("len").desc("My custom len plugin").filter())
    }
    
    fn filter(&mut self, input: Value) -> Result<Vec<ReturnValue>, ShellError> {
        Ok(vec![ReturnSuccess::value(self.len(input)?)])
    }
}
```

Se esse é o caso, por que temos um `begin_filter`? Vamos ver a assinatura do `begin_filter` mais próximo:

```rust
fn begin_filter(&mut self, _: CallInfo) -> Result<Vec<ReturnValue>, ShellError> {
    Ok(vec![])
}
```

Nosso comando `Len` não requer nenhum parâmetro, mas caso precisasse esse seria o local para obtê-los. A partir daqui, podemos configurar nosso filtro, e então usar isso com cada passo do comando `filter` sobre a entrada.

Vamos verificar o próprio `Len` para ver o que ele está fazendo:

```rust
struct Len;

impl Len {
    fn new() -> Len {
        Len
    }

    fn len(&mut self, value: Value) -> Result<Value, ShellError> {
        match &value.value {
            UntaggedValue::Primitive(Primitive::String(s)) => Ok(Value {
                value: UntaggedValue::int(s.len() as i64),
                tag: value.tag,
            }),
            _ => Err(ShellError::labeled_error(
                "Unrecorgnized type in stream",
                "'len' given non-string info by this",
                value.tag.span,
            )),
        }
    }
}
```

Criamos um `Len` muito simples, de fato, que não tem nenhuma estrutura. Ao invés disso é apenas um placeholder que vai permitir a implementação do plugin.

Daqui, criamos dois métodos:

```rust
impl Len {
    fn new() -> Len {
        Len
    }
    // ...
}
```

O primeiro método é opcional: é apenas uma maneira conveniente de criar um novo valor do tipo `Len`. O verdadeiro trabalho é realizado no segundo método:

```rust
impl Len {
    // ...

    fn len(&mut self, value: Value) -> Result<Value, ShellError> {
        match &value.value {
            UntaggedValue::Primitive(Primitive::String(s)) => Ok(Value {
                value: UntaggedValue::int(s.len() as i64),
                tag: value.tag,
            }),
            _ => Err(ShellError::labeled_error(
                "Unrecorgnized type in stream",
                "'len' given non-string info by this",
                value.tag.span,
            )),
        }
    }
}
```

Esse método age sobre cada elemento no pipeline que é recebido pelo nosso plugin. Para o nosso plugin, nos preocupamos apenas com strings para poder retornar o seu tamanho.

Usamos o pattern matching de Rust para verificar o tipo do Value recebido, e entã poderando com ele caso seja uma string. O valor é um Tagged<Value>, então ele armazena com ele de onde o valor surgiu. Se o valor não é um string, retornamos um erro e deixamos o usuário saber de onde veio o valor que está causando o problema. (Note que se quisessemos colocar um erro abaixo do nome do nome do comando, asta basta usar o `name_span` do CallInfo informado no `begin_filter`)

Por último, vamos ver o começo do arquivo:

```rust
use nu_errors::ShellError;
use nu_plugin::{serve_plugin, Plugin};
use nu_protocol::{
    CallInfo, Primitive, ReturnSuccess, ReturnValue, Signature, UntaggedValue, Value,
};
```

Aqui importamos tudo o que precisamos -- tipos e funções -- para ser possível criar nosso plugin.

Assim que acabarmos nosso plugin só precisamos instalá-lo para usá-lo.

```
> cargo install --path .
```

Assim que o `nu` iniciar, vai descobrir o plugin e registrá-lo como um comando.
Se você já estiver executando o `nu` durante o processo de instalação do seu plugin, tenha certeza de que você reiniciou o `nu` para que possa carregar e registrar seu plugin.

```
> nu
> echo hello | len
5
> help len
This is my custom len plugin

Usage:
  > len {flags}

flags:
  -h, --help: Display this help message
```

## Criando um plugin (em Python)

Podemos também criar plugins em outras linguagens de programação. Nessa seção, vamos escrever o mesmo plugin `len` em Python.

Primeiramente, vamos verificar o plugin completo:

```python
#!/usr/bin/python3
import json
import fileinput
import sys


def print_good_response(response):
    json_response = {"jsonrpc": "2.0", "method": "response", "params": {"Ok": response}}
    print(json.dumps(json_response))
    sys.stdout.flush()


def get_length(string_value):
    string_len = len(string_value["item"]["Primitive"]["String"])
    int_item = {"Primitive": {"Int": string_len}}
    int_value = string_value
    int_value["item"] = int_item
    return int_value


for line in fileinput.input():
    x = json.loads(line)
    method = x.get("method", None)
    if method == "config":
        config = {"name": "len", "usage": "Return the length of a string", "positional": [], "named": {}, "is_filter": True}
        print_good_response(config)
        break
    elif method == "begin_filter":
        print_good_response([])
    elif method == "filter":
        int_item = get_length(x["params"])
        print_good_response([{"Ok": {"Value": int_item}}])
    elif method == "end_filter":
        print_good_response([])
        break
    else:
        break
```

Nota: existem maneiras de tornar Python mais robusto, mas aqui deixamos de maneira simples para ajudar com explicações.

Vamos verificar como o plugin funciona, de baixa para cima:

```python
for line in fileinput.input():
    x = json.loads(line)
    method = x.get("method", None)
    if method == "config":
        config = {"name": "len", "usage": "Return the length of a string", "positional": [], "named": {}, "is_filter": True}
        print_good_response(config)
        break
    elif method == "begin_filter":
        print_good_response([])
    elif method == "filter":
        int_item = get_length(x["params"])
        print_good_response([{"Ok": {"Value": int_item}}])
    elif method == "end_filter":
        print_good_response([])
        break
    else:
        break
```

Para esse plugin, temos que For this plugin, we have to serve to basic roles: responding to a request for the plugin configuration and doing the actual filtering. This code acts as our main loop, responding to messages from Nu by doing some work and then returning a response. Each JSON message is sent to the plugin on a single line, so we need only to read the line and then parse the json it contains.

From there, we look at what method is being invoked. For this plugin, there are four methods we care about: config, begin_filter, filter, and end_filter.  When we're sent a 'config' request, we respond with the signature of this plugin, which is a bit of information to tell Nu how the command should be called. Once sent, we break out of the loop so the plugin can exit and be later called when filtering begins.

The other three methods -- begin_filter, filter, and end_filter -- all work together to do the work of filtering the data coming in. As this plugin will work 1-to-1 with each bit of data, turning strings into their string lengths, we do most of our work in the `filter` method. The 'end_filter' method here tells us it's time for the plugin to shut down, so we go ahead and break out of the loop.

```python
def get_length(string_value):
    string_len = len(string_value["item"]["Primitive"]["String"])
    int_item = {"Primitive": {"Int": string_len}}
    int_value = string_value
    int_value["item"] = int_item
    return int_value
```

The work of filtering is doing by the `get_length` function. Here, we assume we're given strings (we could make this more robust in the future and return errors if we were not), and then we extract the string we're given. From there, we measure the length of the string and create a new `Int` value for that length.

Finally, we use the same item we were given and replace the payload with this new Int. We do this to reuse the metadata that was passed to us with the string, though this is an optional step. We could have instead opted to create new metadata and passed that out instead.

```python
def print_good_response(response):
    json_response = {"jsonrpc": "2.0", "method": "response", "params": {"Ok": response}}
    print(json.dumps(json_response))
    sys.stdout.flush()
```

Each response from the plugin back to Nu is also a json message that is sent on a single line. We convert the response to json and send it out with this helper function.

```python
import json
import fileinput
import sys
```

All of this takes a few imports to accomplish, so we make sure to include them.

```python
#!/usr/bin/python3
```

Finally, to make it easier to run our Python, we make this file executable (using something like `chmod +x nu_plugin_len`) and add the path to our python at the top. This trick works for Unix-based platforms, for Windows we would need to create an .exe or .bat file that would invoke the python code for us.

We are using Python 3 because Python 2 will not be maintained past 2020. However script works accordingly with Python 2 and with Python 3.
Just change the first line into: 

```python
#!/usr/bin/python
```

and you are good to go.

## Criando um plugin (em C#)

Você pode aprender mais sobre criar um plugin de Nu com C# aqui:

* .Net Core nu-plugin-lib: (https://github.com/myty/nu-plugin-lib)
