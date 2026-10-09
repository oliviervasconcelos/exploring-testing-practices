# Explorando Práticas de Teste

Neste exercício, vamos explorar práticas de teste em sistemas reais utilizando a ferramenta [TestMiner](https://andrehora.github.io/testminer).

O TestMiner permite visualizar e analisar testes de software em repositórios do GitHub, fornecendo dados sobre como os projetos organizam seus testes, como eles evoluem entre versões e quais bibliotecas de teste são utilizadas.
Explore a ferramenta antes de começar para se familiarizar com seu funcionamento.

Mais detalhes no GitHub da ferramenta: https://github.com/andrehora/testminer.

---

## Passo 1: Selecionar um repositório

Escolha um repositório real que possua testes de software.
Abaixo estão alguns links para ajudá-lo a encontrar projetos interessantes:

- Python: https://github.com/topics/python?l=python
- JavaScript: https://github.com/topics/javascript?l=javascript
- TypeScript: https://github.com/topics/typescript?l=typescript
- Java: https://github.com/topics/java?l=java

- Tópicos: [ai](https://andrehora.github.io/testminer/#topic:ai), [llm](https://andrehora.github.io/testminer/#topic:llm), [api](https://andrehora.github.io/testminer/#topic:api), [nodejs](https://andrehora.github.io/testminer/#topic:nodejs), [android](https://andrehora.github.io/testminer/#topic:android)

- Por organização: [Google](https://andrehora.github.io/testminer/#google), [Microsoft](https://andrehora.github.io/testminer/#microsoft), [Apple](https://andrehora.github.io/testminer/#apple), [Facebook](https://andrehora.github.io/testminer/#facebook), [Netflix](https://andrehora.github.io/testminer/#netflix), 
[GitHub](https://andrehora.github.io/testminer/#github), [Apache](https://andrehora.github.io/testminer/#apache), [HuggingFace](https://andrehora.github.io/testminer/#huggingface)

## Passo 2: Explorar o repositório selecionado

Busque o repositório escolhido no [TestMiner](https://andrehora.github.io/testminer) e analise os dados de teste gerados pela ferramenta.

## Passo 3: Explicar uma prática de teste

Escolha uma prática ou dado de teste relevante e explique com suas próprias palavras.

---

## Instruções de entrega

1. Faça um `fork` deste repositório (saiba mais sobre forks [aqui](https://docs.github.com/pt/pull-requests/collaborating-with-pull-requests/working-with-forks/fork-a-repo)).
2. Responda às questões abaixo diretamente neste arquivo `README.md` do seu fork. Pode adicionar imagens para enriquecer sua explicação.
3. No Moodle, submeta apenas a URL do seu fork.

---

## Respostas

Repositório: https://github.com/pallets/flask

URL TestMiner: https://andrehora.github.io/testminer/#pallets/flask

Explicação:

### Visão geral dos dados

O Flask é um micro-framework web em Python. Segundo a classificação do TestMiner (branch `main`), os arquivos do repositório se dividem assim:

| Categoria     | Arquivos |
|---------------|---------:|
| Source        | 155 |
| Tests         | 36 |
| Test Helpers  | 35 |
| CI Tests      | 1 (`.github/workflows/tests.yaml`) |

Não aparecem arquivos de *mocks*, *snapshots*, *e2e* ou *benchmarks*. Pela visão **Test Location**, quase todos os testes ficam numa pasta `tests/` na raiz, separada do código em `src/flask/`. Cada módulo do framework tem seu arquivo de teste com o prefixo `test_` (`test_blueprints.py`, `test_cli.py`, `test_config.py`, `test_templating.py`...), que é a convenção que o **pytest** usa para encontrar testes automaticamente.

### Prática escolhida: fixtures compartilhadas no `conftest.py`

O que mais me chamou atenção foi o número de *test helpers* (35), quase igual ao de arquivos de teste (36). Isso mostra que o Flask investe bastante em **infraestrutura de teste**, e não só nos testes em si. O centro dessa infraestrutura é o arquivo `tests/conftest.py`, que define **fixtures** do pytest:

```python
@pytest.fixture
def app():
    app = Flask("flask_test", root_path=os.path.dirname(__file__))
    app.config.update(TESTING=True, SECRET_KEY="test key")
    return app

@pytest.fixture
def client(app):
    return app.test_client()
```

Uma fixture é uma função que prepara algo de que o teste precisa (o *setup*). O teste só declara o nome da fixture como parâmetro e o pytest injeta o objeto pronto:

```python
def test_session_path(app, client):
    app.config.update(APPLICATION_ROOT="/foo")
    ...
    rv = client.get("/", "http://example.com:8080/foo")
    assert "path=/foo" in rv.headers["set-cookie"].lower()
```

Com isso, **cada teste recebe uma aplicação nova e isolada**, então um teste não interfere no outro e não é preciso repetir o código de criação da app em todos os arquivos. Em `test_basic.py`, por exemplo, há 90 funções de teste, e 47 delas usam a fixture `client`.

O `conftest.py` também tem duas fixtures `autouse=True`, que rodam automaticamente em **todos** os testes:

- `_standard_os_environ`: limpa variáveis de ambiente como `FLASK_APP` e `FLASK_DEBUG` antes de cada teste, para que o ambiente da máquina não mude o resultado (evita testes "flaky");
- `leak_detector`: depois de cada teste, verifica se algum *app context* ficou aberto e faz o teste falhar se sobrou algum. Ou seja, ele garante que os testes "limpam a sujeira" que deixam.

Os outros *test helpers* que o TestMiner encontrou vão na mesma linha: `tests/test_apps/` tem pequenas aplicações Flask de exemplo (`helloworld`, `cliapp`, `blueprintapp`) usadas para testar o carregamento de apps pela CLI, e `tests/templates/` e `tests/static/` guardam templates e arquivos estáticos usados como dados de entrada nos testes de renderização.

### Uma observação sobre a ferramenta

O TestMiner classifica arquivos por nome, então alguns casos não são exatamente "testes". O `src/flask/testing.py` aparece como teste, mas na verdade é **código de produção**: é o módulo que oferece o `FlaskClient` (`app.test_client()`) aos usuários do framework. Isso também diz algo sobre o projeto: o Flask não só tem testes, ele também **fornece ferramentas de teste** para quem constrói aplicações com ele. É exatamente esse cliente que as fixtures acima usam. Do mesmo jeito, `docs/testing.rst` é documentação sobre como testar apps Flask, e não um teste.

### Conclusão

O Flask usa uma prática bem consolidada no ecossistema Python: testes com pytest numa pasta separada, um arquivo por módulo e *fixtures* centralizadas no `conftest.py`. Essas fixtures dão **reuso** (menos código repetido), **isolamento** (cada teste tem uma app nova e um ambiente limpo) e **verificação automática de vazamentos** entre testes. A quantidade de *test helpers* mostrada pelo TestMiner reflete esse investimento em uma base de testes fácil de manter.
