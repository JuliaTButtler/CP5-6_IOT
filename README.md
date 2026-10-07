# CP5/6_IOT

# RAG — Disruptive Architectures

## Guia para estudo e apresentação

### 1. O que é o projeto?

Nosso projeto implementa um sistema de **RAG — Retrieval-Augmented Generation**, ou **Geração Aumentada por Recuperação**.

A ideia é fazer o modelo de linguagem responder perguntas utilizando como base de conhecimento o **site oficial da disciplina Disruptive Architectures**.

Em vez de simplesmente perguntar algo para o Gemini e deixar que ele responda com o conhecimento que possui, nós fazemos o sistema:

1. acessar o site da disciplina;
2. coletar suas páginas;
3. extrair o conteúdo textual;
4. dividir os conteúdos em trechos menores;
5. transformar esses trechos em embeddings;
6. armazenar esses embeddings em um índice;
7. transformar a pergunta do usuário em um embedding;
8. comparar a pergunta com todos os trechos da base;
9. recuperar os trechos semanticamente mais relevantes;
10. enviar esses trechos para o Gemini como contexto;
11. gerar uma resposta baseada nesse contexto;
12. apresentar também as fontes utilizadas.

A arquitetura geral é:

**Site → documentos → chunks → embeddings → índice vetorial → pergunta → embedding da pergunta → similaridade → Top-K → contexto → Gemini → resposta + fontes**

Esse fluxo é o **pipeline do nosso RAG**.

---

# 2. Por que usar RAG?

Um modelo de linguagem possui conhecimento próprio, mas isso não significa que ele conheça especificamente o conteúdo do nosso site ou que sua resposta esteja necessariamente fundamentada nele.

Por exemplo, se perguntarmos:

> "O que é MQTT?"

O modelo pode saber responder porque MQTT é um conceito conhecido.

Mas isso não significa que ele esteja respondendo **com base no site da disciplina**.

O objetivo do RAG é justamente fornecer ao modelo os conteúdos relevantes da nossa própria base.

Assim:

**Pergunta → recuperar informação relevante → colocar essa informação no contexto → gerar resposta**

Isso ajuda a manter a resposta **grounded**, ou seja, fundamentada na fonte fornecida.

Também permite que o sistema reconheça quando uma informação não está disponível na base.

---

# 3. A base de conhecimento

A primeira decisão do projeto foi definir de onde viriam as informações.

Nossa base de conhecimento é:

**o site oficial da disciplina Disruptive Architectures.**

Não criamos manualmente uma base de textos.

O próprio notebook acessa o site e coleta as páginas listadas no menu da página inicial.

Não é um crawler recursivo: ele não entra em cada página para procurar novos links. Isso é suficiente porque o menu da página inicial já lista as páginas da disciplina.

Isso é importante porque torna o projeto mais automatizado.

Se o site tiver diversas páginas, não precisamos copiar manualmente cada conteúdo para o notebook.

---

# 4. Acessando o site

Primeiro definimos a URL principal:

```python
BASE_URL = "https://arnaldojr.github.io/DisruptiveArchitectures/"
```

Depois fazemos uma requisição HTTP:

```python
response = requests.get(BASE_URL, timeout=10)
response.raise_for_status()
```

### O que está acontecendo?

`requests` permite que o Python faça uma requisição HTTP para o site.

É basicamente o equivalente programático de acessar uma página pela internet.

O:

```python
response.raise_for_status()
```

serve para verificar se a requisição teve sucesso.

Se houver um erro HTTP, como uma página inexistente, o programa gera uma exceção em vez de continuar silenciosamente.

---

# 5. Encontrando as páginas internas

Depois de acessar a página principal, precisamos descobrir quais outras páginas pertencem ao site.

Para isso usamos:

```python
BeautifulSoup
```

Ela permite interpretar o HTML da página.

O código procura os elementos:

```html
<a href="...">
```

que representam links.

Depois utilizamos `urljoin()` para transformar links relativos em URLs completas.

Também filtramos os links para manter apenas páginas que pertencem ao site da disciplina.

O resultado foi:

**28 páginas internas encontradas.**

Essas páginas passam a formar nossos documentos.

---

# 6. Coletando os documentos

Para cada URL encontrada, fazemos uma nova requisição.

Depois o BeautifulSoup extrai o texto da página:

```python
texto = soup.get_text("\n", strip=True)
```

Criamos então uma estrutura contendo:

```python
{
    "url": url,
    "texto": texto
}
```

A URL é muito importante.

Não queremos apenas guardar o conteúdo.

Também queremos saber:

> "De onde veio esse conteúdo?"

Essa informação será utilizada posteriormente para apresentar as fontes da resposta.

No final dessa etapa:

**28 páginas → 28 documentos**

---

# 7. Por que não transformar cada página inteira em um embedding?

Aqui entra uma decisão importante do nosso projeto.

Uma página pode possuir bastante conteúdo.

Se transformássemos cada página inteira em um único vetor, a recuperação ficaria menos granular.

Imagine uma página que fala sobre:

* IoT;
* MQTT;
* sensores;
* arquitetura;
* exemplos;
* exercícios.

Se alguém perguntar especificamente sobre MQTT, recuperar a página inteira não é tão preciso quanto recuperar o trecho que realmente fala sobre MQTT.

Por isso dividimos os documentos em **chunks**.

---

# 8. O que é um chunk?

Um **chunk** é simplesmente um pedaço menor do documento.

Por exemplo:

```text
Documento inteiro
        ↓
┌─────────────────┐
│ chunk 1         │
├─────────────────┤
│ chunk 2         │
├─────────────────┤
│ chunk 3         │
├─────────────────┤
│ chunk 4         │
└─────────────────┘
```

Isso permite que o sistema encontre partes específicas do conteúdo.

Nosso tamanho escolhido foi:

**1000 caracteres**

com:

**200 caracteres de overlap**

---

# 9. O que é overlap?

Overlap significa que dois chunks consecutivos compartilham uma parte do texto.

Exemplo simplificado:

```text
Chunk 1:
AAAA BBBB CCCC DDDD

Chunk 2:
DDDD EEEE FFFF GGGG
```

O trecho `DDDD` aparece nos dois.

Por que fazer isso?

Porque simplesmente cortar um documento a cada 1000 caracteres pode separar uma informação no meio.

O overlap ajuda a preservar contexto entre os chunks.

No nosso projeto:

**tamanho = 1000 caracteres**

**overlap = 200 caracteres**

---

# 10. Resultado da divisão

Depois de dividir as 28 páginas, obtivemos:

**207 chunks**

Cada chunk possui:

* a URL de origem;
* um identificador;
* o texto.

Exemplo conceitual:

```python
{
    "url": "...",
    "chunk_id": 3,
    "texto": "..."
}
```

Agora temos uma base muito mais granular:

**28 documentos → 207 chunks**

---

# 11. O que é um embedding?

Agora chegamos a uma das partes mais importantes do RAG.

Um **embedding** é uma representação numérica de um texto.

Em vez de representar uma frase apenas como palavras, o modelo transforma seu significado em um vetor de números.

Por exemplo, de forma simplificada:

```text
"Como funciona MQTT?"
          ↓
[0.12, -0.43, 0.87, 0.21, ...]
```

Nosso modelo gera vetores com:

**3072 dimensões**

Ou seja, cada chunk é representado por um vetor com 3072 números.

---

# 12. Por que transformar texto em números?

Porque podemos comparar matematicamente esses vetores.

Textos semanticamente parecidos tendem a produzir embeddings próximos em um espaço vetorial.

Por exemplo:

> "O que é MQTT?"

e

> "Como dispositivos trocam mensagens usando MQTT?"

possuem palavras e estruturas diferentes, mas estão semanticamente relacionadas.

O embedding permite que o sistema perceba essa relação.

Isso é o que permite fazer uma **busca semântica**, e não apenas uma busca por palavras exatas.

---

# 13. Embedding dos documentos

Para cada chunk usamos o modelo:

```python
EMBEDDING_MODEL = "gemini-embedding-001"
```

E indicamos, pelo parâmetro `task_type` da API, que aquele conteúdo é um documento para recuperação:

```python
config=types.EmbedContentConfig(task_type="RETRIEVAL_DOCUMENT")
```

Então:

**207 chunks → 207 embeddings**

Esses embeddings formam nosso índice.

---

# 14. Por que usamos processamento em lotes?

Temos 207 chunks.

Em vez de fazer uma chamada à API para cada chunk individualmente, enviamos os conteúdos em lotes.

Por exemplo:

```text
Lote 1 → 50 chunks
Lote 2 → 50 chunks
Lote 3 → 50 chunks
Lote 4 → 50 chunks
Lote 5 → 7 chunks
```

Isso reduz a quantidade de chamadas e torna o processo mais eficiente.

Também implementamos tratamento para erro de quota `429`.

Quando a API informa que a quota foi temporariamente excedida, o programa espera e tenta novamente.

Isso foi importante porque a geração dos 207 embeddings encontrou alguns limites temporários.

No final:

**207/207 processados**

e cada embedding possui:

**3072 dimensões.**

---

# 15. O índice

Depois de gerar os embeddings, armazenamos as informações em `INDICE`.

Cada elemento do índice contém aproximadamente:

```python
{
    "url": "...",
    "chunk_id": 5,
    "texto": "...",
    "embedding": [...]
}
```

Ou seja, temos:

**conteúdo + origem + representação vetorial**

O índice fica em memória durante a execução do notebook.

Não utilizamos uma ferramenta externa como Chroma ou Pinecone.

Isso mantém o projeto simples e próximo da arquitetura apresentada no laboratório da disciplina.

---

# 16. O que acontece quando o usuário faz uma pergunta?

Agora começa a parte de **retrieval**, ou recuperação.

Imagine que o usuário pergunte:

> "O que é MQTT?"

Não podemos comparar diretamente a frase com os números dos documentos.

Primeiro precisamos transformar a própria pergunta em um embedding.

---

# 17. Embedding da pergunta

A pergunta também passa pelo modelo de embeddings:

```text
"O que é MQTT?"
       ↓
embedding da pergunta
       ↓
vetor de 3072 dimensões
```

Usamos:

```python
config=types.EmbedContentConfig(task_type="RETRIEVAL_QUERY")
```

Essa distinção indica que estamos criando um embedding para uma consulta que será usada para procurar documentos.

Agora temos:

**vetor da pergunta**

e

**207 vetores dos chunks**

---

# 18. Similaridade de cosseno

Precisamos descobrir quais chunks são mais próximos da pergunta.

Para isso usamos **similaridade de cosseno**.

A função compara dois vetores e produz um score de similaridade.

De forma simplificada:

* score maior → maior proximidade semântica;
* score menor → menor proximidade semântica.

Por exemplo:

```text
Pergunta
   ↓
Embedding
   ↓
Comparação
   ├── Chunk 1 → 0.82
   ├── Chunk 2 → 0.71
   ├── Chunk 3 → 0.63
   └── Chunk 4 → 0.40
```

O sistema ordena os resultados pelo score.

---

# 19. Uma coisa MUITO importante sobre o score

O score de similaridade **não significa que a resposta está correta**.

Ele indica apenas que os vetores são semanticamente próximos.

Isso é uma distinção importante.

Por exemplo, fizemos uma busca por:

> "O que é IoT?"

e alguns resultados eram bastante relevantes, mas também apareceu um conteúdo apenas relacionado ao tema.

Então:

**similaridade ≠ verdade**

**similaridade ≠ garantia de resposta**

Ela é apenas um sinal de proximidade semântica.

Essa é uma das observações importantes do laboratório do professor.

---

# 20. Top-K

Depois de calcular a similaridade com todos os chunks, ordenamos os resultados.

Selecionamos os melhores.

Isso é o **Top-K**.

No nosso sistema utilizamos:

```python
top_k = 5
```

Ou seja:

> recuperar os 5 chunks com maior similaridade.

Por exemplo:

```text
207 chunks
   ↓
comparação
   ↓
ordenação
   ↓
Top 5
```

Esses cinco chunks serão usados como contexto para o modelo generativo.

---

# 21. Por que não recuperar os 207?

Porque não precisamos mandar todo o site para o modelo.

Isso seria:

* menos eficiente;
* mais caro;
* mais difícil para o modelo processar;
* menos focado.

Queremos fornecer apenas as informações mais relevantes para aquela pergunta.

---

# 22. O que acontece depois da recuperação?

Agora temos:

```text
Pergunta
   ↓
Embedding
   ↓
Similaridade
   ↓
Top 5 chunks
```

Precisamos transformar esses chunks em um **contexto** para o Gemini.

Por exemplo:

```text
Fonte: https://...

[conteúdo do chunk]

Fonte: https://...

[conteúdo do chunk]
```

Esse conjunto de informações é enviado junto com a pergunta.

---

# 23. Generation — geração da resposta

Agora entra o modelo de linguagem.

Utilizamos:

```python
GENERATION_MODEL = "gemini-3.5-flash-lite"
```

O modelo recebe:

1. a pergunta;
2. os chunks recuperados como contexto;
3. instruções de como responder.

A diferença fundamental é que não estamos simplesmente perguntando:

> "Gemini, responda isso."

Estamos dizendo:

> "Aqui estão os trechos recuperados da nossa base. Responda utilizando somente essas informações."

---

# 24. O prompt de segurança do RAG

Nosso prompt estabelece algumas regras importantes.

O modelo deve:

* utilizar somente o contexto recuperado;
* não utilizar conhecimento externo;
* não inventar informações;
* informar quando a informação não estiver disponível;
* pedir informações adicionais quando a pergunta for ambígua;
* nunca fornecer uma informação sem fundamento no contexto;
* responder em português;
* ser objetivo.

A ideia é reduzir as chamadas **alucinações** do modelo.

---

# 25. O que são alucinações?

Em IA generativa, uma alucinação acontece quando o modelo produz uma informação que parece plausível, mas não possui fundamento suficiente.

Por exemplo:

Se perguntarmos:

> "Qual é o procedimento para configurar uma rede 5G privada?"

e o site da disciplina não possuir essa informação, não queremos que o Gemini invente um tutorial.

Queremos:

> "A informação não foi encontrada na base de conhecimento."

Isso é uma característica importante do nosso RAG.

---

# 26. Pergunta ambígua

Também adicionamos uma regra para perguntas que não possuem contexto suficiente.

Por exemplo:

> "Qual é o prazo?"

Essa pergunta não deixa claro:

* prazo de quê?
* qual checkpoint?
* qual atividade?
* qual entrega?

Mesmo que o sistema encontre páginas contendo a palavra "prazo", isso não significa que encontrou a resposta certa.

Por isso, o comportamento desejado é pedir esclarecimento.

Por exemplo:

> "Para qual atividade ou checkpoint você se refere?"

Isso é melhor do que inventar ou escolher arbitrariamente uma interpretação.

---

# 27. Pipeline completo

Todas essas etapas formam o nosso pipeline:

```text
                 BASE DE CONHECIMENTO
                         │
                         ▼
                    Site oficial
                         │
                         ▼
                    Documentos
                         │
                         ▼
                       Chunks
                         │
                         ▼
                     Embeddings
                         │
                         ▼
                  Índice vetorial
                         │
                         │
                         │
Pergunta ───────► Embedding da pergunta
                         │
                         ▼
                Similaridade de cosseno
                         │
                         ▼
                      Top-K
                         │
                         ▼
                     Contexto
                         │
                         ▼
                      Gemini
                         │
                         ▼
                Resposta + Fontes
```

Essa é provavelmente a **melhor figura mental para vocês apresentarem**.

---

# 28. Nossa função `responder`

Criamos uma função que reúne as principais etapas:

```python
def responder(pergunta, top_k=5):
```

Ela recebe a pergunta.

Depois:

### 1. Recupera os chunks

```python
resultados = recuperar_chunks(
    pergunta,
    top_k=top_k
)
```

### 2. Gera a resposta

```python
resposta = gerar_resposta(
    pergunta,
    resultados
)
```

### 3. Mostra a resposta

```text
RESPOSTA
========
...
```

### 4. Mostra as fontes

```text
FONTES
======
- URL 1
- URL 2
```

Assim, temos uma interface simples para executar o pipeline completo.

---

# 29. Por que mostrar as fontes?

As fontes ajudam na **rastreabilidade** da resposta.

O usuário consegue saber quais páginas foram utilizadas pelo sistema.

Isso também facilita a avaliação.

Se o sistema responder alguma coisa estranha, podemos verificar:

> "Quais chunks ele recuperou?"

e:

> "De qual página eles vieram?"

Isso é muito importante em sistemas RAG.

---

# 30. Testes realizados

Não basta implementar o RAG.

Também precisamos avaliar seu comportamento.

Seguimos uma estrutura semelhante à utilizada no laboratório.

Fizemos quatro tipos de teste.

---

# 31. Teste 1 — informação direta

Pergunta:

> **"O que é MQTT?"**

O objetivo é verificar se o sistema encontra diretamente informações sobre MQTT.

O resultado foi positivo.

O sistema recuperou conteúdos relacionados ao MQTT e gerou uma resposta explicando:

* MQTT;
* comunicação;
* publish/subscribe;
* broker;
* tópicos.

Também apresentou as fontes.

---

# 32. Teste 2 — paráfrase

Pergunta:

> **"Como os dispositivos podem trocar mensagens usando MQTT?"**

Aqui não estamos simplesmente repetindo uma pergunta igual à presente no texto.

Estamos reformulando a ideia.

Isso testa uma característica importante dos embeddings:

**busca semântica.**

Mesmo que a formulação da pergunta seja diferente, o sistema consegue encontrar conteúdos relacionados.

Esse é um dos grandes benefícios de usar embeddings em vez de simplesmente procurar palavras iguais.

---

# 33. Teste 3 — informação ausente

Pergunta:

> **"Qual é o procedimento para configurar uma rede 5G privada?"**

A ideia é testar se o RAG vai inventar alguma coisa.

O resultado esperado é:

> **"A informação não foi encontrada na base de conhecimento."**

Esse teste é importante porque demonstra que o sistema não deve tratar qualquer informação relacionada como suficiente para responder.

---

# 34. Teste 4 — pergunta ambígua

Pergunta:

> **"Qual é o prazo?"**

Essa pergunta foi particularmente interessante.

O sistema recuperou páginas relacionadas a prazos (agenda e checkpoints), mas a pergunta não especificava de qual atividade ou checkpoint estava falando.

Em vez de escolher um prazo qualquer, o modelo **pediu que o usuário especificasse a atividade**. Esse é o comportamento esperado, e vem da regra que adicionamos ao prompt:

> Se não for possível identificar com segurança o que o usuário está perguntando, pedir informações adicionais.

Esse teste mostra que **recuperar algo semanticamente relacionado não significa necessariamente que temos contexto suficiente para responder corretamente.**

---

# 37. Limitações do sistema

Nosso RAG funciona, mas não é perfeito.

Algumas limitações são importantes de mencionar.

### 1. Similaridade não garante resposta correta

O chunk mais parecido pode ser apenas relacionado ao assunto.

### 2. Top-K fixo

Estamos usando `K=5`.

Dependendo da pergunta, cinco chunks podem ser:

* informação demais;
* informação de menos;
* ou conter alguns resultados irrelevantes.

O próprio laboratório mostra que testar diferentes valores de `K` é uma possibilidade.

### 3. Perguntas ambíguas

O sistema pode encontrar algum contexto relacionado mesmo quando a pergunta não possui informação suficiente.

Por isso adicionamos a instrução para pedir esclarecimento.

### 4. Base dependente do site

O RAG só consegue responder com segurança sobre aquilo que está disponível na base coletada.

Se uma informação não estiver no site, o sistema não deve inventá-la.

### 5. Índice em memória

Nosso índice é criado durante a execução do notebook.

Isso é adequado para o projeto didático, mas um sistema de produção poderia utilizar um banco ou banco vetorial persistente.

---

# 38. O que NÃO fizemos

Também é importante saber explicar nossas escolhas.

Nós **não utilizamos**:

* LangChain;
* Chroma;
* Pinecone;
* banco vetorial externo;
* arquitetura complexa de agentes;
* múltiplos modelos;
* reranking;
* infraestrutura de produção.

Isso foi intencional.

O laboratório trabalha diretamente com:

**embeddings + índice + similaridade + recuperação + geração.**

Então implementamos essas partes diretamente em Python para deixar claro o funcionamento interno do RAG.

---

# 39. O que adaptamos em relação ao laboratório?

Nosso projeto segue a mesma ideia do Lab 4, mas possui uma adaptação importante.

No laboratório, a base utilizada é pequena e os chunks são escritos manualmente.

No nosso projeto, a base é o site inteiro da disciplina.

Por isso precisamos automatizar:

```text
Site
 ↓
Links
 ↓
Páginas
 ↓
Texto
 ↓
Chunks
```

Depois seguimos a mesma lógica:

```text
Chunks
 ↓
Embeddings
 ↓
Índice
 ↓
Recuperação
 ↓
Contexto
 ↓
Geração
```

Essa adaptação faz sentido porque nosso problema é maior que a base de exemplo do laboratório.

---

# 40. Por que a nossa divisão automática de chunks faz sentido?

Essa foi uma das nossas principais adaptações.

Em vez de escrever manualmente:

```text
chunk 1
chunk 2
chunk 3
...
```

para todas as páginas, criamos uma função que divide automaticamente cada documento.

Isso permite transformar:

**28 documentos → 207 chunks**

sem precisar preparar manualmente cada trecho.

Isso também se relaciona ao desafio de chunking automático apresentado no próprio laboratório.

---

# 41. Por que usamos processamento em lotes nos embeddings?

Também foi uma adaptação prática.

Temos 207 chunks.

Fazer uma chamada separada para cada chunk seria desnecessariamente pesado.

Por isso usamos lotes de até 50 chunks.

Além disso, adicionamos tratamento para o erro de quota `429`.

Isso não muda o conceito do RAG.

Apenas torna a implementação mais adequada para a quantidade de conteúdo que estamos processando.

---

# 42. A diferença entre retrieval e generation

Essa distinção é muito importante para a apresentação.

### Retrieval

É a parte que responde:

> "Quais informações da base são relevantes para essa pergunta?"

Ela envolve:

* embedding da pergunta;
* comparação com embeddings;
* similaridade;
* ranking;
* Top-K.

### Generation

É a parte que responde:

> "Como transformar essas informações recuperadas em uma resposta compreensível?"

Ela utiliza o Gemini.

Portanto:

**Retrieval encontra.**

**Generation escreve.**

---

# 43. A diferença entre embedding e geração

Também não devemos confundir os dois.

### Embedding

Transforma texto em vetor numérico.

```text
Texto → vetor
```

Serve para comparação semântica.

### Modelo generativo

Recebe texto/contexto e produz texto.

```text
Contexto + pergunta → resposta
```

No nosso projeto usamos os dois:

**Embedding → recuperar**

**Gemini generativo → responder**

---

# 44. A diferença entre chunk e documento

Um documento é a página coletada.

Um chunk é uma parte desse documento.

No nosso projeto:

```text
1 página
   ↓
vários chunks
```

Cada chunk mantém a URL de origem para que saibamos de onde ele veio.

---

# 45. A diferença entre índice e banco de dados

Nosso `INDICE` não é um banco de dados tradicional.

É uma estrutura em memória que guarda os chunks e seus embeddings.

Ela funciona como um índice vetorial simples para o experimento.

Em um sistema de produção poderíamos utilizar soluções especializadas, mas isso não é necessário para demonstrar o funcionamento do RAG.

---

# 46. Como explicar tudo em poucos minutos

Se o professor pedir:

> "Explique o projeto."

Uma resposta possível é:

> "Nosso projeto implementa um RAG utilizando o site oficial da disciplina como base de conhecimento. Primeiro coletamos as páginas internas do site utilizando requisições HTTP e BeautifulSoup. Depois extraímos o texto e dividimos os documentos em chunks com overlap. Cada chunk é transformado em um embedding utilizando o modelo do Gemini e armazenado em um índice junto com sua URL de origem.
>
> Quando o usuário faz uma pergunta, também transformamos essa pergunta em um embedding e calculamos a similaridade de cosseno entre ela e os embeddings dos chunks. Recuperamos os cinco chunks mais relevantes e usamos esses trechos como contexto para o Gemini. O modelo então gera uma resposta baseada somente nesse contexto e mostramos também as fontes utilizadas.
>
> Nos testes avaliamos informação direta, paráfrase, informação ausente e ambiguidade. Os testes mostraram que o sistema consegue recuperar informações relevantes e reconhecer quando uma informação não está disponível, embora ainda existam limitações em perguntas ambíguas e na própria recuperação semântica."

---

# 47. Se perguntarem "onde está a IA?"

A IA aparece principalmente em duas partes.

### 1. Embeddings

O modelo de embedding transforma os textos em representações vetoriais que permitem realizar busca semântica.

### 2. Geração

O Gemini utiliza os chunks recuperados como contexto para gerar a resposta.

Portanto:

```text
IA para representar → embeddings

IA para responder → modelo generativo
```

---

# 48. Se perguntarem "por que não perguntar diretamente ao Gemini?"

Porque o objetivo do RAG é fornecer uma **base específica e controlada** para a resposta.

Sem RAG:

```text
Pergunta → Gemini → resposta
```

Com RAG:

```text
Pergunta
   ↓
Buscar informações relevantes na base
   ↓
Contexto
   ↓
Gemini
   ↓
Resposta fundamentada
```

O segundo fluxo permite controlar melhor de onde as informações utilizadas na resposta vieram.

---

# 49. Se perguntarem "RAG elimina alucinações?"

Não.

O RAG **reduz o risco**, mas não garante que nunca haverá erro.

Por isso implementamos:

* recuperação;
* contexto;
* instruções explícitas;
* reconhecimento de ausência;
* pedido de esclarecimento;
* apresentação das fontes;
* testes.

Mesmo assim, a recuperação pode trazer conteúdo inadequado ou o modelo pode interpretar o contexto incorretamente.

---

# 50. Se perguntarem "por que usar similaridade de cosseno?"

Porque precisamos comparar os embeddings.

A similaridade de cosseno mede a proximidade entre os vetores considerando principalmente sua direção no espaço vetorial.

No nosso caso:

```text
embedding da pergunta
        ↕
embedding do chunk
```

Quanto maior a similaridade, maior a proximidade semântica indicada pelo modelo.

---

# 51. Se perguntarem "por que Top-K = 5?"

Não existe um número universalmente correto.

Escolhemos 5 para fornecer alguns trechos relevantes ao modelo sem enviar toda a base.

O próprio laboratório mostra que `k=1`, `k=3` e `k=5` podem ser comparados.

Então `K=5` é uma escolha experimental, não uma regra do RAG.

---

# 52. Se perguntarem "o que acontece se o documento correto não estiver no Top-K?"

Então o modelo generativo provavelmente não terá a informação necessária no contexto.

Esse é um problema de **retrieval**.

Mesmo que o Gemini saiba a resposta por conhecimento próprio, nosso prompt orienta que ele **não utilize conhecimento externo**.

Portanto, o sistema deve preferir admitir que não encontrou a informação a inventar ou completar com conhecimento externo.

---

# 53. O ponto principal do projeto

O mais importante é entender que o projeto não é simplesmente:

> "um chatbot que responde perguntas."

Ele é um pipeline de:

**recuperação + contexto + geração.**

A parte de recuperação é justamente o diferencial do RAG.

O modelo não recebe simplesmente uma pergunta.

Ele recebe:

```text
Pergunta
+
Informações recuperadas da base
```

E utiliza essas informações para gerar a resposta.

---

# 54. Resumo de tudo em uma frase

Se vocês esquecerem todo o resto durante a apresentação, lembrem desta:

> **"Nosso RAG coleta o conteúdo do site da disciplina, transforma os textos em embeddings, recupera semanticamente os trechos mais relevantes para cada pergunta e utiliza esses trechos como contexto para o Gemini gerar uma resposta fundamentada nas fontes da própria base."**

---

# 55. Mapa mental final

```text
                         RAG
                          │
          ┌───────────────┴───────────────┐
          │                               │
     CONSTRUIR BASE                  RESPONDER
          │                               │
          ▼                               ▼
       Site                           Pergunta
          │                               │
          ▼                               ▼
      Documentos                     Embedding
          │                               │
          ▼                               ▼
       Chunks                     Similaridade
          │                               │
          ▼                               ▼
     Embeddings                       Top-K
          │                               │
          ▼                               ▼
        Índice                         Contexto
                                          │
                                          ▼
                                       Gemini
                                          │
                                          ▼
                                  Resposta + fontes
```

## O fluxo que vocês precisam lembrar

**1. Coletamos o site.**

**2. Extraímos os documentos.**

**3. Dividimos em chunks.**

**4. Geramos embeddings.**

**5. Criamos o índice.**

**6. Recebemos uma pergunta.**

**7. Geramos o embedding da pergunta.**

**8. Calculamos similaridade de cosseno.**

**9. Pegamos os Top-K chunks.**

**10. Montamos o contexto.**

**11. O Gemini gera a resposta.**

**12. Mostramos as fontes.**

**13. Avaliamos o comportamento com testes.**

Esse é o projeto inteiro.
