---
layout: post
title: "Como gerar mapa de fluxos automaticamente no QGIS"
categories: [SIG, QGIS]
tags: [sig, qgis, camada virtual, sql, geotecnologias]
---


Os mapas de fluxo, em sistemas de informação geográfica, constituem um recurso muito utilizado para analisar os deslocamentos de pessoas e cargas entre áreas ou pontos de um território. No QGIS3, é possível gerar esses mapas dinamicamente através de uma camada virtual. Com algumas linhas de consulta SQL, configuramos uma camada que vetoriza automaticamente as linhas que representarão os fluxos e que ainda se atualizará dinamicamente se os dados de fluxo ou as geometrias de referência forem alteradas. Vejamos como funciona.

![No QGIS3 é possível criar mapas de fluxo com camada virtual](https://paulovitorweb.files.wordpress.com/2020/11/flow-maps-qgis3-thumbnail.png?w=698)
_No QGIS3 é possível criar mapas de fluxo com camada virtual_

Neste exemplo, eu vou utilizar os dados da *Matriz de Origem/Destino real de deslocamentos de pessoas elaborada a partir de Big Data da telefonia móvel*, apresentada e mantida pela Secretaria Nacional de Aviação Civil do Ministério da Infraestrutura, disponível [aqui](https://dados.gov.br/dataset/matriz-de-origem-destino-real-de-deslocamentos-de-pessoas-por-big-data-da-telefonia-movel).

Vou utilizar três camadas:

1. aeroportos: a camada de aeroportos, que contém um campo chamado ICAO com o código de cada aeroporto;
2. fluxo: uma tabela com os dados dos fluxos com embarque no Aeroporto Presidente Castro Pinto (código SBJP), na Paraíba, no mês de janeiro, que também contém campos com os códigos dos aeroportos de embarque e desembarque. Utilizaremos esses campos para fazer a relação;
3. Uma camada dos estados brasileiros, como base, para melhor ilustração.

Aqui:

![Camada dos estados brasileiros](https://paulovitorweb.files.wordpress.com/2020/11/flow-maps-qgis3-1.png?w=563)

A consulta:

```sql
SELECT f.aerodromoembarque, f.aerodromodesembarque, f.fluxo,
        make_line(a1.geometry, a2.geometry)
 FROM fluxo f
 JOIN aeroportos a1 ON f.aerodromoembarque = a1.ICAO
 JOIN aeroportos a2 ON f.aerodromodesembarque = a2.ICAO
```

![Consulta para criar linhas para o mapa de fluxos](https://paulovitorweb.files.wordpress.com/2020/11/flow-maps-qgis3-2.png?w=554)
_Consulta para criar linhas para o mapa de fluxos_

Estamos fazendo duas uniões importantes com a mesma camada, a de aeroportos, duplicando-a (a1 e a2): em uma, fazemos a união pelo código do aeroporto de embarque (a1); na outra, pelo de desembarque (a2). A função `make_line` faz o trabalho de desenhar uma linha ligando cada par de origem/destino.

O resultado:

![Resultado](https://paulovitorweb.files.wordpress.com/2020/11/flow-maps-qgis3-3-1.png?w=459)

Agora é só estilizar conforme sua necessidade. Por exemplo, buscando melhor representação, abaixo classifiquei por espessura da linha em 7 classes, utilizando quebras naturais, e apliquei transparência.

![Resultado com novo estilo](https://paulovitorweb.files.wordpress.com/2020/11/flow-maps-qgis3-4-2.png?w=507)

É isso. Rápido, prático e escalável.

Espero que ajude. Até a próxima.