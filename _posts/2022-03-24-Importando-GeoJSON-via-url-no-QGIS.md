---
layout: post
title: "Importando GeoJSON via url no QGIS"
img_path: /assets/img/
categories: [SIG, QGIS]
tags: [sig, qgis, geojson, camada vetorial, geotecnologias]
---

O [GeoJSON](https://tools.ietf.org/html/rfc7946){:target="_blank"} é um formato de intercâmbio de dados geoespaciais baseado no formato JSON – JavaScript Object Notation. É um recurso muito interessante para quem trabalha com dados espaciais, principalmente pelo vasto uso do JSON para comunicação entre sistemas. É possível, por exemplo, disponibilizar recursos geográficos simples numa URL com atualização regular para aplicações e usuários que precisam do dado atualizado.

No QGIS, é possível importar um arquivo GeoJSON disponível em URL na internet. Tão simples quanto importar um shapefile local.

1. Ir em Adicionar Camada Vetorial;
2. Na guia Vetor, no lugar da opção Arquivo, marcar Protocolo: HTTP(s)…;
3. Em Protocolo, escolher Tipo GeoJSON;
4. Em URI, inserir a URL do geojson.

Como na imagem abaixo, em que importo os [bairros de João Pessoa](https://github.com/paulovitorweb/geodata-jp){:target="_blank"} no QGIS3.

![Adicionando camada vetorial a partir de um GeoJSON hospedado na internet](adding-layer-from-geojson.png)
_Adicionando camada vetorial a partir de um GeoJSON hospedado na internet_

É isso.

![Camada de bairros de João Pessoa adicionada no QGIS3](added-layer.png)
_Camada de bairros de João Pessoa adicionada no QGIS3_

Se você quer conhecer mais sobre o formato GeoJSON, o [RFC 7946](https://tools.ietf.org/html/rfc7946){:target="_blank"}  é o documento técnico de referência.

Até a próxima.