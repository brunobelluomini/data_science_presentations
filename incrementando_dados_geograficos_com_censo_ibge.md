# Incrementando dados geográficos com o Censo Nacional do IBGE

Geolocalização é um ponto importante em diversos domínios. Serviços de entrega, seguradoras ou mesmo pesquisas para saber se uma nova unidade de uma franquia dará resultado estão muito interessados em saber mais características sobre uma determinada região.

Como um dos produtos da Creditas é o empréstimo com garantia de imóvel, a localização é fator importante.

No contexto de ciência de dados, saber CEP, bairro, cidade e estado podem não ser o suficiente. O correto é encontrarmos outras informações mais genéricas que expliquem características daquela localidade.

Digamos que um modelo de machine learning foi treinado com uma feature que recebe o nome do bairro em um dataset contendo apenas exemplos de clientes do estado de São Paulo. Se este modelo receber algum cliente do Rio de Janeiro para fazer uma predição ele não vai saber o que fazer com um bairro de valor “Barra da Tijuca” e ou vai quebrar ou resultar numa predição completamente equivocada.

Em contrapartida, se o modelo receber a quantidade de habitantes entre 18 e 60 anos daquele mesmo bairro, isso generaliza o fator de localização geográfica para o modelo uma vez que qualquer lugar pode ter o mesmo número contabilizado.

**Portanto, busque sempre usar features como `renda_per_capita_média` ou `índice_de_furtos_a_cada_100_habitantes` em vez do nome do Bairro ou Cidade, por exemplo.**

_”Mas onde conseguir estes dados?”_ você se pergunta. Existem algumas fontes de dados abertas como o [Geosampa](http://geosampa.prefeitura.sp.gov.br/PaginasPublicas/_SBC.aspx) para a cidade de São Paulo, mas numa escala nacional uma das melhores para se utilizar é o *censo nacional do IBGE*.

## Sobre o Censo Nacional do IBGE
O Censo Nacional do IBGE é o principal estudo estatístico sobre a população brasileira e ocorre a cada 10 anos, sendo a versão mais recente realizada
em 2010. Nele é possível encontrar informações como número de pessoas alfabetizadas, maiores de idade, infra estrutura urbana, etc.

Os dados coletados são em função de uma unidade territorial chamada **setor censitário**. Existem cerca de 314 mil setores censitários no Censo 2010 mapeados em todo o Brasil. Cada setor censitário possui um id único no formato `<UF><MMMMM><DD><SD><SSSS>`, onde:

```
UF – Unidade da Federação
MMMMM – Município
DD – Distrito
SD – Subdistrito
SSSS – Setor
```

![](/images/setores_censitarios_vila_olimpia.png)
Exemplo de organização de setores censitários na cidade de São Paulo. Cada setor é limitado pelas linhas azuis mais escuras.


A base de dados coletados do Censo IBGE 2010 pode ser encontrada [aqui](https://www.ibge.gov.br/estatisticas-novoportal/downloads-estatisticas.html) seguindo o caminho:

```
Censos -> Censo_Demografico_2010 -> Resultados_do_Universo -> Agregados_por_Setores_Censitarios
```

Os arquivos de dados possuem encoding `iso8859_15` e `;` como separador.

## Chega de papinho, hora da prática
Sem mais delongas, vamos a um exemplo prático considerando o endereço da Creditas:

```Python
import pandas as pd

geolocation_info_df = pd.DataFrame({
    ‘address’: ['Avenida Engenheiro Luis Carlos Berrini'],
    ‘address_number’: ['105'],
    ‘address_city’: ['São Paulo'],
    ‘address_state’: ['SP']
})
```
| address | address_number | address_city | address_state |
|----------------------------------------|-----|-----------|----|
| Avenida Engenheiro Luis Carlos Berrini | 105 | São Paulo | SP |


Nosso objetivo é extrair as variáveis do Censo para este endereço. Quando abrir a base de dados do Censo vai se deparar com algo similar à tabela abaixo:

| Cod_setor       |...| V001 | V002 | V003 |...|
|-----------------|---|------|------|------|---|
| 355030835000017 |...| 160  | 500  | 3,13 |...|


`Cod_setor` é código do setor censitário e `V001`, `V002` e `V003` são exemplos das variáveis encontradas. As explicações sobre elas podem ser encontradas na documentação do Censo também disponível no link para a base de dados.

O desafio então vai ser transformar a string do endereço no código `355030835000017` para extrair as variáveis. A ligação entre as duas será feita usando um arquivo geoespacial chamado Shapefile (`.shp`), um dataframe que contém características geográficas cujo formato é parecido com isto:

|ID      | CD_GEOCODI    |...| geometry                                     |
|--------|---------------|---|----------------------------------------------|
| 115583 |355030835000017|...|POLYGON((-46.69188 -23.602605, -46.691487 -23…|

As duas colunas que importam para nós são a `CD_GEOCODI`, o código do setor censitário, e a `geometry` que contém um objeto `POLYGON` com as coordenadas geográficas (longitude e latitude respectivamente) do desenho daquele setor.

A ideia do processo vai ser de converter o endereço em longitude e latitude para então pegar o código do setor com o Shapefile.

Os arquivos Shapefile `.shp` dos setores censitários podem ser acessados [aqui](http://geoftp.ibge.gov.br/organizacao_do_territorio/malhas_territoriais/malhas_de_setores_censitarios__divisoes_intramunicipais/censo_2010/setores_censitarios_shp/).


## Convertendo endereço para Lat/Lon

Existem diversos serviços que permitem fazer a conversão do endereço em longitude e latitude. Neste exemplo vamos utilizar a API do Google Maps [Geocoding](https://developers.google.com/maps/documentation/geocoding/intro) junto com a biblioteca [`googlemaps`](https://github.com/googlemaps/google-maps-services-python).

Para utilizar esta API é necessária uma chave de autenticação. Você pode obtê-la para fazer algumas requests gratuitamente [aqui](https://developers.google.com/maps/documentation/geocoding/get-api-key).

```Python
import googlemaps

gmaps_client = googlemaps.Client(key=gmaps_api_key)
gmaps_response = gmaps.geocode(address='Avenida Engenheiro Luis Carlos Berrini, 105, São Paulo, SP')

gmaps_response_location = gmaps_response[0]['geometry']['location']
longitude = gmaps_response_location['lng']
latitude = gmaps_response_location['lat']

print(f'{longitude}, {latitude}')
# -46.6899982, -23.5984886
```

## Obtendo o código do setor censitário do um endereço

Agora que conseguimos o ponto com as coordenadas nós precisamos encontrar em qual setor censitário ele está localizado. O próximo passo é converter a string de coordenadas num objeto `Point` para que o Shapefile entenda que se trata de um ponto geográfico. Vamos usar a biblioteca `shapely` para isso.

```Python
from shapely.geometry import Point

geolocation_info_df['coordinate_point'] = Point(longitude, latitude)
geolocation_info_df
```

| address                                | ... | coordinate_point               |
|----------------------------------------|-----|--------------------------------|
| Avenida Engenheiro Luis Carlos Berrini | ... | Point(-46.6899982 -23.5984886) |

O próximo passo é carregar o Shapefile dos setores censitários com a biblioteca `geopandas` para fazermos o match do ponto com o polígono do setor.

```Python
import geopandas as gpd

census_sector_gpd = gpd.read_file(
'sp_setores_censitarios/35SEE250GC_SIR.shp'
)

geolocation_info_df['census_code'] = geolocation_info_df['coordinate_point'].map(
    lambda x: census_sector_gpd.loc[census_sector_gpd.contains(x), 'CD_GEOCODI'].values
).str[0].astype('inf64')
geolocation_info_df
```

| address                                | ... | census_code     |
|----------------------------------------|-----|-----------------|
| Avenida Engenheiro Luis Carlos Berrini | ... | 355030835000017 |

Finalmente, o último passo é dar um merge do nosso dataset com o que contém as variáveis do censo e correr pro abraço.

```Python
ibge_census_features_df = pd.read_csv(
    'Database/SP1/CSV/Basico_SP1.csv',
    encoding='iso8859_15',
    sep=';'
)

geolocation_info_with_ibge_df = geolocation_info_df.merge(
    ibge_census_features_df,
    left_on='census_code',
    right_on='Cod_setor'
)

geolocation_info_with_ibge_df
```

O resultado final ficará assim:

| address                                | ... | V001 | V002 | V003 |...|
|----------------------------------------|-----|------|------|------|---|
| Avenida Engenheiro Luis Carlos Berrini | ... | 160  | 500  | 3,13 |...|

Boas! Você acabou de enriquecer sua base de dados com características do endereço que vão te permitir fazer predições em qualquer lugar do Brasil =D

