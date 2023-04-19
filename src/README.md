## Introdução ao DynamoDB

### :computer: Iniciando o AWS CLI

Para iniciar o AWS CLI via linha de comando, basta digitarmos no prompt de comando:

```bash
$ aws
```

Se o CLI ainda não estiver configurado, é necessário usar o comando

```bash
$ aws configure	
```

e inserir uma **chave de acesso** e uma **chave de acesso secreta**, as quais podem ser geradas pelo site do AWS fazendo login com sua conta de usuário, além da região e do formato padrão de arquivos de saída, que pode ser o JSON. Para o caso de usuários situados no Brasil, a região a ser inserida deve ser ``sa-east-1``.



### :bar_chart: Trabalhando com o banco de dados

#### :pencil: Criando a tabela

Para **criar** uma tabela chamada *Music* no AWS DynamoDB, primeiramente, criamos um arquivo JSON com o nome ``create-table-music.json`` no seguinte formato:

```json
{
    "TableName": "Music",
    "AttributeDefinitions": [
        {
            "AttributeName": "Artist",
            "AttributeType": "S"
        },
        {
            "AttributeName": "SongTitle",
            "AttributeType": "S"
        }
    ],
    "KeySchema": [
        {
            "AttributeName": "Artist",
            "KeyType": "HASH"
        },
        {
            "AttributeName": "SongTitle",
            "KeyType": "RANGE"
        }
    ],
    "ProvisionedThroughput": {
        "ReadCapacityUnits": 10,
        "WriteCapacityUnits": 5
    }
}
```

Em seguida, acessamos, através do CLI, o diretório onde este arquivo está e criamos a tabela digitando o seguinte comando:

```bash
$ aws dynamodb create-table --cli-input-json file://create-table-music.json
```



#### :pencil: Inserindo dados

Para **adicionar um** item à tabela, criamos um arquivo JSON com o nome ``insert-data-table.json`` e o preenchemos com o seguinte código:

````json
{
    "TableName": "Music",
    "Item": {
        "Artist": {"S": "Iron Maiden"},
        "SongTitle": {"S": "Chains of Misery"},
        "AlbumTitle": {"S": "Fear of the Dark"},
        "SongYear": {"S": "1992"}
    }    
}
````

Depois de salvar o arquivo, com o prompt de comando no mesmo diretório, digitamos o comando:

````bash
$ aws dynamodb put-item --cli-input-json file://insert-data-table.json
````

Para **adicionar múltiplos itens** à tabela, criamos um script JSON chamado ``insert-multiple-data-table.json`` no modelo

````json
{
    "RequestItems": {
        "Music": [
            {
                "PutRequest": {
                    "Item": {
                        "Artist": {"S": "Iron Maiden"},
                        "SongTitle": {"S": "Wasting Love"},
                        "AlbumTitle": {"S": "Fear of the Dark Live"},
                        "SongYear": {"S": "1992"}
                    }
                }
            },
            {
                "PutRequest": {
                    "Item": {
                        "Artist": {"S": "Iron Maiden"},
                        "SongTitle": {"S": "Weekend Warrior"},
                        "AlbumTitle": {"S": "Fear of the Dark"},
                        "SongYear": {"S": "1992"}
                    }
                }
            },
            {
                "PutRequest": {
                    "Item": {
                        "Artist": {"S": "Iron Maiden"},
                        "SongTitle": {"S": "Fear of the Dark"},
                        "AlbumTitle": {"S": "Fear of the Dark Tour"},
                        "SongYear": {"S": "1992"}
                    }
                }
            }
        ]
    }    
}
````

e o salvamos. Em seguida, fazemos a criação da tabela digitando o seguinte comando no prompt:

````bash
$ aws dynamodb batch-write-item --cli-input-json file://insert-multiple-data-table.json
````

#### :pencil: Pesquisando dados

Para **pesquisar itens por artista**, podemos criar um JSON com nome ``search-by-artist.json``, inserir:

````json
{
    "TableName": "Music",
    "KeyConditionExpression": "Artist = :artist",
    "ExpressionAttributeValues": {
        ":artist": {"S":"Iron Maiden"}
    }
}
````

e executá-lo no prompt de comando:

````bash
$ aws dynamodb query --cli-input-json file://search-by-artist.json
````

No entanto, se quisermos fazer a pesquisa tendo como base o título do álbum, por exemplo, neste momento, não conseguiremos. Para que isso seja possível, precisamos **criar um índex global secundário que seja baseado neste parâmetro**, ou seja, em ``AlbumTitle``. Fazemos isso através de um update na tabela. 

Assim como para os passos anteriores, vamos criar um JSON. Desta vez, o chamaremos de ``update-table-sec-index-album-title.json`` e o preencheremos com:

````json
{
    "TableName": "Music",
    "AttributeDefinitions": [
        {
            "AttributeName": "AlbumTitle",
            "AttributeType": "S"
        }
    ],
    "GlobalSecondaryIndexUpdates": [
        {
            "Create": {
                "IndexName": "AlbumTitle-index",
                "KeySchema": [
                    {
                        "AttributeName": "AlbumTitle",
                        "KeyType": "HASH"
                    }
                ],
                "ProvisionedThroughput": {
                    "ReadCapacityUnits": 10,
                    "WriteCapacityUnits": 5
                },
                "Projection": {
                    "ProjectionType": "ALL"
                }
            }
        }
    ]
}
````

Para executar o update, usamos o comando:

````bash
$ aws dynamodb update-table --cli-input-json file://update-table-sec-index-album-title.json
````

Agora podemos finalmente **pesquisar na nossa tabela utilizando o título do álbum**. Para isso, basta alterarmos algumas linhas do JSON de busca anterior, aquele referente ao artista. Para manter o histórico de códigos, vamos criar um novo arquivo, chamado ``search-by-album-title.json``. O script nesse arquivo será:

````json
{
    "TableName": "Music",
    "IndexName": "AlbumTitle-index",
    "KeyConditionExpression": "AlbumTitle = :name",
    "ExpressionAttributeValues": {
        ":name": {"S":"Fear of the Dark"}
    }
}
````

A consulta é feita usando o mesmo comando de pesquisa anterior, modificando-se apenas o nome do arquivo:

````bash
$ aws dynamodb query --cli-input-json file://search-by-album-title.json
````

Vamos agora **criar um índice secundário com dois parâmetros: o artista e o nome do álbum**. Para isso, basta adicionar algumas linhas ao arquivo JSON anterior e, obviamente, modificar o nome do índice para que não sejam criados dois iguais. Criamos um novo arquivo, chamado ``update-table-sec-index-artist-album-title.json`` e inserimos o código:

````json
{
    "TableName": "Music",
    "AttributeDefinitions": [
        {
            "AttributeName": "Artist",
            "AttributeType": "S"
        },
        {
            "AttributeName": "AlbumTitle",
            "AttributeType": "S"
        }
    ],
    "GlobalSecondaryIndexUpdates": [
        {
            "Create": {
                "IndexName": "ArtistAlbumTitle-index",
                "KeySchema": [
                    {
                        "AttributeName": "Artist",
                        "KeyType": "HASH"
                    },
                    {
                        "AttributeName": "AlbumTitle",
                        "KeyType": "RANGE"
                    }
                ],
                "ProvisionedThroughput": {
                    "ReadCapacityUnits": 10,
                    "WriteCapacityUnits": 5
                },
                "Projection": {
                    "ProjectionType": "ALL"
                }
            }
        }
    ]
}
````

gerando, assim, um índice de nome ``ArtistAlbumTitle-index``, enquanto o anterior era ``AlbumTitle-index``.

O comando para executar o script é o mesmo de anteriormente:

````bash
$ aws dynamodb update-table --cli-input-json file://update-table-sec-index-artist-album-title.json
````

Conseguimos agora **pesquisar pelas músicas utilizando o nome do artista e do álbum**, através do arquivo ``search-by-artist-album-title.json``:

````json
{
    "TableName": "Music",
    "IndexName": "ArtistAlbumTitle-index",
    "KeyConditionExpression": "Artist = :v_artist and AlbumTitle = :v_title",
    "ExpressionAttributeValues": {
        ":v_artist": {"S":"Iron Maiden"},
        ":v_title": {"S": "Fear of the Dark"}
    }
}
````

Fazemos a busca com o comando de buscas usual:

````bash
$ aws dynamodb query --cli-input-json file://search-by-artist-album-title.json
````

Por último, vamos **criar um index global secundário que seja baseado no nome da música e no ano**. O processo é exatamente o mesmo feito anteriormente, mas vamos adicioná-lo. Primeiro, o JSON de update, ``update-table-sec-index-song-title-year.json``:

````json
{
    "TableName": "Music",
    "AttributeDefinitions": [
        {
            "AttributeName": "SongTitle",
            "AttributeType": "S"
        },
        {
            "AttributeName": "SongYear",
            "AttributeType": "S"
        }
    ],
    "GlobalSecondaryIndexUpdates": [
        {
            "Create": {
                "IndexName": "SongTitleYear-index",
                "KeySchema": [
                    {
                        "AttributeName": "SongTitle",
                        "KeyType": "HASH"
                    },
                    {
                        "AttributeName": "SongYear",
                        "KeyType": "RANGE"
                    }
                ],
                "ProvisionedThroughput": {
                    "ReadCapacityUnits": 10,
                    "WriteCapacityUnits": 5
                },
                "Projection": {
                    "ProjectionType": "ALL"
                }
            }
        }
    ]
}
````

O comando de execução se modifica apenas no nome do arquivo:

````bash
$ aws dynamodb update-table --cli-input-json file://update-table-sec-index-song-title-year.json
````

A **pesquisa** também é feita de forma exatamente igual, com o JSON, ``search-by-song-title-year.json``:

````json
{
    "TableName": "Music",
    "IndexName": "SongTitleYear-index",
    "KeyConditionExpression": "SongTitle = :v_song and SongYear = :v_year",
    "ExpressionAttributeValues": {
        ":v_song": {"S":"Wasting Love"},
        ":v_year": {"S": "1992"}
    }
}
````

e utilizando o comando

````bash
$ aws dynamodb query --cli-input-json file://search-by-song-title-year.json
````