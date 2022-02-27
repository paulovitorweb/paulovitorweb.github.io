---
layout: post
title: "Modelagem e mapeamento automáticos de um shapefile no GeoDjango"
categories: [Python, Django]
tags: [python, django, geodjango, geotecnologias]
---


Um recurso muito útil e que pode economizar tempo e trabalho quando você está fazendo a modelagem de dados geográficos no GeoDjango é automatizar a escrita do modelo e do dicionário de mapeamento.

Já fiz essa escrita de forma manual algumas vezes, mas hoje digo seguramente que, independentemente da quantidade de campos, é muito melhor automatizar o processo, fazendo apenas um ou outro ajuste antes de executar a migração e a importação do shape.

Você pode fazer isso com o `ogrinspect`. A sintaxe é essa:

```bash
python manage.py ogrinspect [options] <data_source> <model_name> [options]
```

No exemplo abaixo, vamos gerar de forma automática o modelo e o meapeamento de um shape de polígonos. São os bairros da cidade de João Pessoa. O shape está disponível [aqui](http://geo.joaopessoa.pb.gov.br/digeoc/htmls/donwloads.html). Você precisa executar o comando na pasta raiz do seu projeto. Para manter as coisas organizadas, crie uma pasta data dentro da pasta do seu aplicativo e coloque os arquivos lá. Neste exemplo, a pasta do aplicativo é `geo`. No terminal, execute:

```bash
python manage.py ogrinspect geo\data\Bairros.shp Bairro --srid=4326 --mapping
```

A opção `–srid` define o sistema de referência espacial. A opção `–mapping` informa que queremos também o dicionário de mapeamento, para usarmos com o `LayerMapping`. O retorno deve ser algo parecido com isso:

```python
from django.contrib.gis.db import models

class Bairro(models.Model):
    bairro = models.CharField(max_length=100)
    cód = models.IntegerField()
    cod_loc = models.FloatField()
    bairro_1 = models.CharField(max_length=254)
    densidade = models.FloatField()
    perim_km = models.FloatField()
    area_km2 = models.FloatField()
    geom = models.PolygonField(srid=4326)

# Auto-generated `LayerMapping` dictionary for Bairro model
bairro_mapping = {
    'bairro': 'BAIRRO',
    'cód': 'CÓD',
    'cod_loc': 'Cod_loc',
    'bairro_1': 'Bairro_1',
    'densidade': 'Densidade',
    'perim_km': 'Perim_km',
    'area_km2': 'Area_km2',
    'geom': 'POLYGON',
}
```

É isso. Espero que seja útil.

Até mais.