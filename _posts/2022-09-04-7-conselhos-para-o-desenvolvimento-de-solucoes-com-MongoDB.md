---
layout: post
title: "7 conselhos para o desenvolvimento de soluções com MongoDB"
image: /assets/img/mongodb-tips-thumbnail.png
categories: [Banco de dados, MongoDB]
tags: [banco de dados, nosql, database, mongodb, mongo]
---

Depois de passar alguns anos usando apenas bancos de dados relacionais e há mais de um ano trabalhando com MongoDB, passei de alguém que torcia o nariz quando o assunto era bancos de dados não relacionais para alguém que entende o seu propósito e como bancos de dados orientados a documento como o MongoDB podem agregar valor tanto para o desenvolvimento quanto para o negócio.

Ao longo desse tempo fui tomando nota de alguns insights e resolvi reunir alguns deles nesse texto, em que elenco 7 conselhos que daria para alguém que está iniciando algum projeto com MongoDB ou mesmo quem já trabalha com ele há algum tempo. Eles podem ajudar a tomar boas decisões desde a modelagem até a construção de consultas.

Espero que seja útil.

E, claro: críticas e sugestões são bem-vindas.

São eles:

1. Use documentos e matrizes incorporados. O MongoDB por muito tempo não ofereceu suporte a transações justamente por não ser uma preocupação primária, afinal a possibilidade de construir um modelo de dados aproveitando incorporações tornava o suporte a transações desnecessário em muitos casos práticos, já que a alteração de um único documento é atômica, e é possível traduzir o modelo normalizado de bancos relacionais em um documento único no MongoDB.

    - Por exemplo, se você tem uma coleção de usuários que podem ter até três endereços cadastrados, considere incorporar os documentos de endereço em um array dentro do documento de usuário. Isso vai evitar a necessidade de buscar essa informações em outra coleção e, em algumas situações, tornar transações desnecessárias, pois a alteração de endereço e de outros dados do usuário será, por definição, atômica;
    
    - Também é possível explorar o ganho semântico de usar documentos incorporados. Por exemplo, a partir do mesmo exemplo, se cada usuário puder ter apenas um endereço, você pode ainda assim incorporar os dados em um documento dentro do campo de endereço em vez de espalhá-los na raiz do documento, a menos que você tenha um motivo para isso. É mais semântico acessar os dados com `endereco.cidade` do que usar nomes de campos como `endereco_cidade` ou `enderecoCidade`. Além disso, se por algum motivo não for desejável trazer os dados de endereço, apenas não projetar o campo `endereco` vai desconsiderar tudo que há dentro dele. Ainda, se a aplicação evoluir de forma que deixe de ser interessante armazenar o endereço incorporado na coleção de usuário, vai ser mais fácil fazer a migração para uma coleção separada ou até outro banco (relacional ou não), pois os nomes dos campos poderão permanecer iguais. Semântica importa.
    
    - Outro exemplo: suponhamos que você tenha uma tabela de posts em que cada post pode ter uma ou mais tags. Neste caso, faz total sentido armazenar as tags em um array dentro do documento de cada post. Ex: `tags: ["bancodedados", "mongodb"]`.
    
    Além disso, é importante considerar que normalmente o custo de desempenho de se gravar em um único documento é menor do que a transação de vários documentos.
    
    De qualquer modo, sempre é necessário refletir sobre a melhor escolha para cada caso específico. Nem sempre incorporar é a melhor opção. O que nos leva ao próximo conselho.
    
2. O fato de que o MongoDB suporta incorporações e que você pode aplicar isso em diversos níveis de um documento não quer dizer que você deve resumir todo seu banco de dados em uma única coleção. Em outras palavras, não leve a expressão “não relacional” tão ao pé da letra.
    
    Relações simples entre documentos em coleções separadas com frequência podem ser o melhor design. Por exemplo, não faz sentido incorporar todas as postagens de um usuário de uma rede social dentro do documento de usuário. Além de essa matriz incorporada poder crescer para além do razoável, é possível pensar em uma série de situações em que você precisará alterar ou buscar as postagens mas não o usuário. Ou ainda precisar de índices em campos dos documentos de posts. De todo modo, com o id indexado na coleção de usuários e armazenado em cada documento na coleção de postagens, você pode buscar a informação de que precisa através de um `$lookup` com a rapidez e performance garantida pelo índice. Não se assuste se vir bancos de dados MongoDB com dez, vinte, trinta coleções. Ou até mais.
    
    “Ah, mas isso não seria transformar o Mongo em um banco relacional?” Bem, você quem pode dizer. No Mongo é possível usar relações. Mas e em um MySQL, por exemplo, você pode usar a estrutura desnormalizada do Mongo? Use a ferramenta certa para o problema certo sem se apegar muito a que ferramenta é “dona” desse ou daquele paradigma de armazenamento de dados. Sempre vão aparecer novas soluções que unem o melhor de dois conceitos diferentes - e, ainda assim, perdem em algum quesito.
    
3. A forma como o MongoDB usa objetos BSON para executar consultas fornece uma excelente camada de segurança contra os ataques tradicionais de injeção de SQL - você pode encontrar uma explicação sucinta para isso [aqui](https://www.mongodb.com/docs/v6.0/faq/fundamentals/#bson){:target="_blank"}. Mas existe a possibilidade de usar expressões que executam operações JavaScript arbitrárias diretamente no servidor que podem resultar em vulnerabilidade se não forem utilizadas adequadamente. São elas: `$where`, `mapReduce`, `$accumulator`, `$function`. O ideal é evitar usar essas operações, até porque em muitos casos práticos você pode expressar todas as necessidades de consulta de uma aplicação sem usar JavaScript. 
    
    De todo modo, você também pode desabilitar todas as execuções de JavaScript no lado do servidor com uma configuração simples descrita [aqui](https://www.mongodb.com/docs/v6.0/faq/fundamentals/#javascript){:target="_blank"}. É uma boa prática desabilitar por padrão tudo aquilo que você não precisa e que pode resultar em algum problema de segurança.
    
4. Pode ser tentador usar o estágio `project` em uma pipeline para diminuir o tráfego de dados e economizar na largura de banda de rede, mas é preciso ter em mente que isso adicionará mais processamento à consulta e quase sempre resultará num tempo de resposta maior. Reflita sempre sobre os prós e contras de seu uso e sobre o que trará mais valor a um caso específico. Um princípio razoável - que, obviamente, não deve ser levado em consideração isoladamente - é usar `project` para tasks assíncronas quando for desejável e evitar em consultas síncronas.

5. Há uma exceção para o descrito acima. Uma boa combinação entre índices e projeção pode tornar uma busca muito mais rápida através de uma “consulta coberta”. 
    
    Por exemplo, imagine que você precise apenas dos ids dos documentos cujo campo `tipo` seja igual a um determinado valor. Se você adicionar um índice ao campo `tipo` e, na consulta, usar esse campo como filtro e adicionar um estágio `project` que traga apenas o `id` de cada documento - que já é indexado por padrão -, o mongo fará uma consulta coberta, o que significa que ele não precisará buscar os documentos, pois tudo que ele precisa (para buscar e retornar) está nos índices. [Este artigo](https://betterprogramming.pub/improve-mongodb-performance-using-projection-c08c38334269){:target="_blank"} mostrou que uma consulta coberta pode ser até 250 vezes mais rápida do que uma sem índices.
    
6. As consultas que você precisará fazer na sua aplicação determinam que modelo de dados é o ideal. É importante que você entenda com que frequência e quais dados a sua aplicação mais precisará inserir/buscar/alterar para que possa, então, desenhar o melhor modelo. Boa parte da performance de uma aplicação na camada do banco de dados é resultado de uma boa modelagem, com os índices certos nos lugares certos, e com incorporações sempre que fizerem sentido, e evitando incorporações onde não trouxerem ganho.

7. Por fim: o MongoDB não é bala de prata. Esse é um cliché bem conhecido no mundo da tecnologia, mas é sempre importante lembrar. 
    
    - Primeiro de tudo: dada a possibilidade de inserir documentos em coleções sem obedecer a um schema (embora você possa contornar isso tanto no nível do cliente quanto no nível do próprio mongo), é relativamente fácil as coisas saírem do controle se o time não tiver experiência o suficiente para lidar com essa responsabilidade. Sem o devido cuidado, você pode começar a ter problemas ao ler documentos como, esperando inteiros, receber texto; ou, esperando sempre um documento, receber um valor nulo. O tio Ben já dizia: com grandes poderes vêm grandes responsabilidades. Use clientes com schemas para validar as operações de E/S;
    
    - O MongoDB tem um excelente desempenho para escrita, mas [costuma perder](https://www.mongodb.com/compare/mongodb-mysql){:target="_blank"} na hora de ler uma grande quantidade de registros em comparação com bancos relacionais. Mesmo que você possa alcançar uma excelente performance na leitura através de índices e de consultas cobertas, se sua aplicação precisa com frequência ler mil, dez mil, cem mil registros, e esses registros podem ficar contidos em uma ou no máximo duas tabelas relacionadas com índices, talvez você deva considerar armazenar esses dados em um banco relacional. Para essa necessidade um MySQL ou PostgreSQL pode ser a melhor escolha, tudo vai depender de cada caso. Se você puder criar um microsserviço para essa necessidade específica que use um banco relacional e deixar o mongo armazenar outros dados que precisem de mais escrita, pode acabar sendo o melhor dos mundos;
    
    - Bancos relacionais foram feitos para ter um ótimo desempenho quando se precisa de relacionamentos entre tabelas. Os índices e a integridade referencial entre elas garantem a consistência e uma boa performance, mesmo que você precise de duas, três ou mais tabelas para buscar dados que na ponta do processo vão ser um único objeto. Ao elaborar o modelo de dados de uma solução, você pode se deparar com a constatação de que ele é inerentemente relacional. Está lá: você vê que precisa dos dados relacionados tanto juntos quanto, frequentemente, separados. Você precisa de consultas otimizadas por índices em muitos níveis, ou seja, em grande parte das tabelas. Você percebe que a leitura de uma quantidade massiva de dados importa tanto quanto a escrita. É possível ver com clareza os esquemas das tabelas, e a possibilidade de precisar escrever muitas migrações ao longo da via útil da aplicação para ajustar os esquemas rígidos é remota. Todas esses são indícios de que você pode estar diante de um caso em que o melhor caminho é seguir com um banco relacional.
