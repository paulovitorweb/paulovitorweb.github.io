---
layout: post
title: "SQL espacial e otimização: ligando pontos a linhas pela proximidade"
categories: [Banco de dados, PostGIS]
tags: [banco de dados, postgresql, postgis, geotecnologias]
---


Um tempo atrás precisei encontrar uma forma de relacionar um grande volume de geometrias pela localização no PostGIS. Tinha uma base com centenas de milhares de pares de coordenadas de GPS de veículos e uma malha de logradouros extraídas do OpenStreetMap em que cada feição era um segmento da via, e precisava responder a seguinte pergunta:

Dado um par de coordenadas registrado pelo GPS, em que rua ele está?

![Pontos e ruas associados](https://paulovitorweb.files.wordpress.com/2022/02/spatial-sql-paulovitorweb.gif?w=424)
_Precisava associar cada ponto a uma linha pela proximidade_

Preferencialmente, essa consulta deveria consumir a menor quantidade de recursos e executar no menor tempo possível. Solucionar esse problema me fez assimilar alguns conceitos muito importantes de consultas espaciais no PostGIS e acabei revisitando essa semana enquanto revia algumas anotações de estudo.

Bom, vamos lá.

Primeiro, precisei gerar uma geometria do tipo ponto a partir das coordenadas GPS, que estavam no formato geográfico decimal:

```sql
ST_Transform(ST_SetSRID(ST_MakePoint(long, lat), 4326), 31985)
```

Basicamente, criamos um ponto com `ST_MakePoint`, depois definimos o sistema de referência espacial usando o código EPSG com `ST_SetSRID` e, por fim, transformamos para o sistema métrico, no qual a base de logradouros também estava.

Ambas as bases tinham índices espaciais.

Ok. Vamos à primeira consulta que pensei:

```sql
SELECT g.id gid, o.id oid, o.name logradouro
  FROM gps g, jp_osm o
  WHERE ST_Contains(ST_Buffer(o.geom, 10), g.geom)
```

Essa consulta apresenta alguns problemas:

- Demorou muito para executar, não é performática;
- `ST_Buffer` não é uma boa ideia para seleção espacial por proximidade pois não gera um buffer exato, apenas a aproximação de um;
- A consulta acaba associando cada ponto a mais de um logradouro.

Usar `ST_Distance` para filtrar também não seria uma boa ideia porque ela não pode se aproveitar dos índices espaciais. Ela faria uma varredura para encontrar todas as combinações de distância e isso seria muito, muito lento.

Então, depois de alguns testes e leitura de documentação, cheguei a essa consulta:

```sql
SELECT DISTINCT ON (g.id) g.id gid, o.id oid, o.name logradouro
   FROM gps g
   LEFT JOIN jp_osm o ON ST_DWithin(o.geom, g.geom, 10)
   WHERE o.id IS NOT NULL
   ORDER BY g.id, ST_Distance(g.geom, o.geom)
```

Ela encontra a linha mais próxima de cada ponto dentro de uma tolerância de 10 metros, ou seja, acha o segmento de via mais provável em que cada veículo estava em cada registro do posicionamento terrestre. Essa consulta se mostrou superior como solução do problema que tinha pelos seguintes motivos:

- Executa em menos de 3% do tempo da outra consulta, ou seja, muito mais performática;
- Elimina duplas associações. Com a cláusula DISTINCT ON, que retorna apenas a primeira linha de um grupo distinto definido como parâmetro, é selecionada apenas a primeira linha com id de GPS distinto – `DISTINCT ON (g.id)` – e, ao ordenar pela menor distância entre o ponto e cada logradouro – `ST_Distance(g.geom, o.geom)` -, isso significa que apenas a associação com a linha mais próxima é selecionada;
- Usa `ST_DWithin`, que pode usar índices e é mais precisa para a seleção de geometrias dentro de uma certa distância do que a combinação `ST_Contains` + `ST_Buffer`;

Resolver esse problema acabou sendo muito útil não só pela finalidade, também pelo fato de que aprendi muitos conceitos importantes para consultas espaciais.

Espero que seja útil pra mais alguém que veja isso e precise solucionar um problema parecido.