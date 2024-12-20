# Relatório Final

## Objetivos

Criar um Chatbot inteligente integrado ao banco de dados da IDGeo, onde um usuário faça perguntas em linguagem natural e obtenha uma resposta também em linguagem natural, condizente com a sua pergunta e os dados obtidos.

## Abordagem

Tomamos uma abordagem onde criamos 3 princiais funções, que posteriormente adicionaremos a uma função maior.

![Chart]([https://github.com/user-attachments/assets/0639f702-5572-4235-8731-21c8eb886be2])

### Tradutor (Linguagem Natural - SQL)

Essa função utiliza um modelo LLM, e deve receber como entrada um comando/pergunta em linguagem natural, e retornar uma Query SQL correspondente àquela pergunta. Adicionalmente, pode receber também como parâmetro um contexto com mensagens e dados prévios da conversa.

### Busca no DB

Essa função deve fazer a busca propriamente dita no banco de dados. Deve receber como parâmetro uma Query SQL, executar a query no DB, e retornar os dados obtidos como resposta. Não utiliza modelos LLM para isso.

### Tradutor (Dados - Linguagem Natural)

Essa função deve pegar os dados obtidos na busca anterior, a pergunta feita pelo usuário e o histórico de conversa, e gerar com o modelo LLM uma resposta em linguagem natural com base no contexto dos dados fornecidos. 

### Chatbot

Função de chatbot padrão, onde quando o usuário perguntar sobre algo que não é relacionado ao banco de dados, ele será direcionado.

## Testes de resposta à perguntas de baixa/média complexidade do usuário

O programa respondeu muito bem aos testes realizados com perguntas simples, tais como:

-Quantos marcadores foram criados no mês de outubro/22 na região de interesse 646?

-Quais são os principais diagnósticos do mês de outubro/22?

-Quantos diagnósticos contendo daninhas foram criados no mês de outubro/22?

-Qual é a data de criação do marcador 1954?

-Quais são as principais informações do marcador 1954?



Testamos 20 perguntas, e dessas 20, somente 6 precisaram ser reescritas de forma que funcionassem corretamente.

## Testes de resposta à perguntas mais complexas do usuário

### 1. Quais os diagnósticos apontados nos marcadores presentes na parcela MB034-01$1?

Solução esperada:

```sql
select diagnostico1_atual, diagnostico2_atual from marcadores where parcela = 'MB034-01$1';
```

Ou

```sql
select diagnostico1_atual from marcadores where diagnostico1_atual is not null and parcela = 'MB034-01$1'
union
select diagnostico2_atual from marcadores where diagnostico2_atual is not null and parcela = 'MB034-01$1';
```

Solução entregue pelo programa:

```sql 
SELECT diagnostico1_atual, diagnostico2_atual 
FROM marcadores 
WHERE parcela = 'MB034-01$1';
```

O programa nesse caso nos retornou exatamente a Query desejada (1 opção), tendo assim gerado a seguinte resposta:

```
Os diagnósticos apontados nos marcadores presentes na parcela MB034-01$1 são:

1. **Daninhas - Folha Larga**
2. **None** (nenhum outro diagnóstico adicional foi identificado).

Se precisar de mais informações ou orientações sobre como lidar com as daninhas de folha larga, estou à disposição para ajudar!
```

### 2. Qual foi a região de interesse com o maior número de alertas de nível de atenção Alto?

Solução esperada:

```sql
select regioes_de_interesse.nome, count(marcadores.id) from marcadores
join regioes_de_interesse on (regioes_de_interesse.id = marcadores.regiao_de_interesse_id)
where tipo_atual = 'Alerta' and nivel_atencao_atual = 'Alto'
group by regiao_de_interesse_id
order by count(marcadores.id) desc
limit 1;
```

Solução entregue pelo programa:

```sql
SELECT regioes_de_interesse.nome, COUNT(marcadores.id) AS num_alertas
FROM regioes_de_interesse
JOIN marcadores ON regioes_de_interesse.id = marcadores.regiao_de_interesse_id
WHERE marcadores.nivel_atencao_atual = 'Alto'
GROUP BY regioes_de_interesse.nome
ORDER BY num_alertas DESC
LIMIT 1;
```

O programa, nesse caso, não levou em consideração o fato de termos que analizar somente pontos com `tipo_atual='Alerta'`, e nos gerou uma Query que retorna a seguinte resposta (errada):

```
A região de interesse com o maior número de alertas de nível de atenção Alto foi Inhambupe, com um total de 64 alertas.
```

Refazendo a pergunta e deixando mais explicito o erro que ele cometeu (_Qual foi a região de interesse com o maior número de alertas (tipo atual Alerta) de nível de atenção Alto?_), tivemos a seguinte Query retornada pelo programa:

```sql
SELECT regioes_de_interesse.nome, COUNT(marcadores.id) AS numero_de_alertas
FROM marcadores
JOIN regioes_de_interesse ON marcadores.regiao_de_interesse_id = regioes_de_interesse.id
WHERE marcadores.tipo_atual = 'Alerta' AND marcadores.nivel_atencao_atual = 'Alto'
GROUP BY regioes_de_interesse.nome
ORDER BY numero_de_alertas DESC
LIMIT 1;
```

Nesse caso, temos uma Query extremamente parecida com a esperada, tendo somente algumas alterações na lógica da busca, nos retornando a seguinte resposta (agora correta):

```
A região de interesse com o maior número de alertas de nível de atenção Alto é Alagoinhas, com um total de 6 alertas.
```

### 3. Qual o diagnóstico mais frequente no período do mês 10/22 ao mês 11/22?

Query esperada:

```sql
select diagnostico, count(*)
from (
	select diagnostico1_atual as diagnostico from marcadores where date(criacao) between '2022-10-01' and '2022-11-30'
	union all
	select diagnostico2_atual from marcadores where date(criacao) between '2022-10-01' and '2022-11-30'
)
where diagnostico is not null
group by diagnostico
order by count(*) desc
limit 1
```

Solução entregue pelo programa:

```sql
SELECT diagnostico1_atual, COUNT(*) as frequencia 
FROM marcadores 
WHERE criacao BETWEEN '2022-10-01' AND '2022-11-30' 
GROUP BY diagnostico1_atual 
ORDER BY frequencia DESC 
LIMIT 1;
```

O programa falha em criar uma Query que analise tanto o `diagnostico1_atual` quanto o `diagnostico2_atual`, lendo assim somente metade dos diagnósticos. No entanto, a resposta final estava correta, pois o caso com maior ocorrência só ocorreu no campo `diagnostico1_atual`.

```
Com base no conjunto de dados fornecido, o diagnóstico mais frequente no período do mês 10/22 ao mês 11/22 é "Daninhas - Folha Larga", com uma frequência de 42 ocorrências.
```

Para corrigir esse problema, foi tentada a mesma solução usada para a pergunta anterior (alterar a pergunta para esclarecer o erro), mas em nenhum dos testes o programa corrigiu seu erro, exceto quando foi especificado exatamente como deveria ser feito. Esse problema poderia ser solucionado futuramente incluindo um contexto maior para a LLM e/ou adicionando perguntas e respostas corretas ao contexto.

### 4. Quais foram os usuários mais ativos entre outubro/22 e novembro/22?

Query esperada:

```sql
select usuarios.id, count(distinct marcadores_situacoes.marcador_id) as atividades from usuarios
join marcadores_situacoes on (marcadores_situacoes.criador_id = usuarios.id)
and date(marcadores_situacoes.criacao) between '2022-10-01' and '2022-11-30'
group by usuarios.id
order by atividades desc;
```

Query obtida no programa:

```sql
SELECT criador_id, COUNT(*) as atividades
FROM marcadores
WHERE criacao BETWEEN '2022-10-01' AND '2022-11-30'
GROUP BY criador_id
ORDER BY atividades DESC;
```

Essa é uma pergunta subjetiva, pois não é especificado o que é uma "atividade" de um usuário. No caso esperado, é considerada atividade o número de pontos criados ou editados no período, considerando as tabelas `marcadores` e `marcadores_situações`. Já a resposta do programa leva em consideração somente a tabela `marcadores`, gerando a seguinte resposta final:

```
Com base no conjunto de dados fornecido, os usuários mais ativos entre outubro/22 e novembro/22 foram:

1. criador_id: 75, com 98 atividades.
2. criador_id: 99, com 5 atividades.

Os demais usuários (criador_id: 95, criador_id: 114, e criador_id: 216) tiveram apenas 1 atividade cada um. Portanto, o criador_id: 75 foi o mais ativo, seguido pelo criador_id: 99.
```

### 5. Quantos alertas foram transformados em marcadores no mês de outubro/22?

Query esperada:

```sql
select count(distinct marcador_id)
from (
	select marcador_id, criacao, tipo,
	lead(criacao) over (partition by marcador_id order by criacao) as next_criacao,
	lead(tipo) over (partition by marcador_id order by criacao) as next_tipo
	from marcadores_situacoes
)
where tipo = 'Alerta'
and next_tipo = 'Padrão'
and strftime('%Y-%m', next_criacao) = '2022-10'
```

Query obtida:

```sql
SELECT COUNT(*) 
FROM marcadores 
WHERE criacao BETWEEN '2022-10-01' AND '2022-10-31' AND tipo_atual = 'alerta';
```

Essa é uma pergunta complexa. O programa não entendeu corretamente a questão e criou uma Query muito rasa, que não faz a busca que precisamos, nos retornando 0 alertas transformados em marcadores em outubro/22.
Para tentar solucionar essa questão, a pergunta foi reescrita de algumas formas, porém o programa não conseguiu chegar na resposta correta em nenhuma delas, retornando números como 51 ou 33 (resposta esperada: 31).


### Conclusão

Os testes acima foram feitos com perguntas propositalmente difíceis e com a intenção de confundirem a LLM, porém, mesmo assim, o programa teve um bom desempenho médio, conseguindo responder corretamente metade das questões (ainda que algumas depois de uma correção textual por parte do usuário). Isso se deve a alguns fatores, principalmente a baixa descrição do banco de dados e contexto "raso" e o baixo uso de engenharia de prompt em um nível mais avançado. No entanto, ao ser testado com perguntas mais simples que gerariam Querys com menor complexidade (baixa e média complexidade), o programa teve uma ótima eficácia, com cerca de 90% de acerto.
Algumas formas de solucionar esses problemas serão propostas na seção [Perspectivas de Futuro](#perspectivas-de-futuro), mas se somente for aumentada a descrição do banco de dados, já será vista uma grande diferença no resultado final.

## Diferença de modelos e precififação

### Precificação da OpenAI

A precificação da API da OpenAI varia de modelo a modelo, mas todos eles têm dois valores, um de input (tudo que a LLM vai **ler**) e um de output (tudo que a LLM vai **escrever**). Esses valores são geralmente dados a cada 1k ou 1M de *tokens* utilizados.
Um *token* é um pedaço de uma palavra, usado para o processamento de linguagem natural por uma LLM, e cada um equivale a aproximadamente 4 caracteres ou 0,75 palavras. Caso necessário, a própria OpenAI disponibiliza uma plataforma ([Tokenizer Tool](https://beta.openai.com/tokenizer)), onde podem ser calculados os tokens com maior precisão.

### GPT-3.5 Turbo

Acaba não sendo vantajoso, pois sua versão mais barata é mais cara e com um desempenho inferior ao `GPT 4o-mini`

### GPT-4o mini

Melhor opção custo-benefício. Tem um custo médio de US$0,60 por 1M token - output, e de US$0,15 1M token - input. É indicado para lidar com a parte conversacional do chatbot.

### GPT-4o

Opção com melhor custo benefício para fazer a parte de conversão de linguagem natural para Query SQL. Custo médio de US$10,00 de 1M token - output e US$2,50 de 1M token - input.

### OpenAI-o1 mini

Opção nova, recém lançada pela OpenAI, ainda a ser testada no projeto. É a solução da OpenAI para geração e interpretação de código, lógica e problemas matemáticos. É a versão mais barata do `OpenAI-o1`, com o modelo `mini` custando US$12,00 o 1M token - output (incluindo tokens usados para raciocício interno do modelo, que não são dados na saída da API), e US$3,00 o 1M token - input. Sua versão convencional tem um valor mais alto, custando US$60,00 cada 1M token - output e US$15,00 o 1M token - input, e é a melhor solução apresentada pela OpenAI para raciocínio lógico.

### Outros modelos mais antigos

Não compensam por questão financeira. A OpenAI propósitamente deixa os modelos antigos mais caros que os modelos mais novos, para induzir a compra dos modelos novos.

### Fine Tuning de modelo, treinamento ou uso de algum modelo novo

A maior parte das opções exige um esforço bem maior de desenvolvimento do modelo (treinamento de modelo ou uso de modelo com open source, etc), incluindo um grande investimento nessa parte, não sendo 100% garantido que o modelo superará consideravelmente aqueles já treinados. Porém, há uma boa perspectiva de que se possa fazer um fine tuning (aperfeiçoamento e treinamento específico de um modelo pré existente e treinado com dados próprios) de um modelo já existente da própria OpenAI para que seja mais focado no propósito e que possa ter um desempenho superior com um investimento menos significativo. 
No entanto, essa opção deve ser avaliada com mais cautela no futuro, pois envolve um investimento considerável, em qualquer forma a ser feita.

## Perspectivas de Futuro

As perspectivas para o futuro do projeto são muito positivas, dado que já temos um protótipo funcional com uma boa eficácia, e que pode ser melhorada aplicando diversas técnicas e algumas alterações.

### LangChain

Uma das principais alterações a serem feitas é a utilização da plataforma [LangChain](https://www.langchain.com/), que será usada para a criação de um Vector Store Database, substituindo a necessidade de termos que criar uma descrição detalhada do banco de dados (que não é uma solução 100%), fazendo com que a LLM tenha conhecimento direto do formato, tipos de dados, colunas, tabelas, ligações e etc do DB. O uso do LangChain no futuro irá impulsionar abruptamente a eficácia do programa, principalmente em casos de perguntas mais complexas e que exigem um melhor conhecimento do bando de dados.
No entanto, teremos o desafio de achar uma forma de trabalhar com o banco de dados da IDGeo, que está hospedado na plataforma Datasette (que não possui uma ligação direta com o LangChain) e uma forma de atualizar o Vector Store DB para incluir novas informações gastando o mínimo possível de poder computacional.

### Engenharia de prompt

O estudo dessa nova ciência vai permitir que criemos contextos mais abrangentes e que conduzam melhor as respostas da LLM para aquilo que buscamos, sem fugir do escopo desejado ou fornecer uma resposta não condizente com a pergunta.

### Alteração da temperatura dos modelos

Uma alteração que não fizemos testes por ser algo muito abrangente é a temperatura dos modelos. A temperatura, basicamente, dita o grau de liberdade e "criatividade" que o modelo tem para fornecer alguma resposta. Em nossos testes, fizemos testes somente com `temperatura = 0`, mas podem ser testados outros valores para se decidir qual atende melhor as necessidades de cada função no programa.

### Estudo de maneiras de salvar o histórico de conversa

Atualmente, estamos salvando a conversa inteira no histórico. No entanto, para conversas mais longas, essa é uma péssima opção, pois estaremos aumentando substancialmente o input para o modelo, aumentando tanto o valor, como o tempo de espera para processamento. Além disso, quanto mais informações tivermos no input do modelo, maior a chance de ele se confundir e usar informações desnecessárias.
Temos algumas formas de resolver esses problemas, como por exemplo criar um limite de caracteres ou tokens armazenados na memória de mensagens, ou um limite de quantidade de mensagens em memória. Além disso, para manter a precisão e fidelidade ao tópico abordado pelo Chatbot, podemos também usar pontos chaves (assim como o ChatGPT faz atualmente), para fornecer esses dados como contexto.
