# gerenciador-de-catalago

Vamos construir um **gerenciador de catálogos da Netflix** usando o **Azure Functions** e o **CosmosDB** (banco de dados NoSQL), juntamente com o **Azure Storage Account** para armazenar arquivos, como imagens e vídeos. Esse projeto envolverá várias etapas, e vou dividir o processo em partes práticas para cada uma das funcionalidades que você mencionou.

### **1. Criando a Infraestrutura na Nuvem**

Primeiro, vamos criar os recursos no **Azure** para a aplicação. Você pode usar o **Azure Portal**, mas para manter tudo em código, podemos usar o **Azure CLI**.

#### **Passo 1: Criar um Grupo de Recursos**
```bash
az group create --name NetflixCatalogGroup --location eastus
```

#### **Passo 2: Criar uma Conta de Armazenamento (Storage Account)**

A conta de armazenamento vai ser usada para armazenar arquivos (como imagens de capas dos filmes).

```bash
az storage account create --name netflixcatalogstorage --resource-group NetflixCatalogGroup --location eastus --sku Standard_LRS
```

#### **Passo 3: Criar um Banco de Dados Cosmos DB**

Para armazenar informações sobre os filmes, vamos usar o **CosmosDB**, que oferece escalabilidade e baixa latência.

```bash
az cosmosdb create --name NetflixCatalogDB --resource-group NetflixCatalogGroup --kind MongoDB
```

#### **Passo 4: Criar uma Função Azure (Azure Function App)**

Crie um App de Função, que será onde as Azure Functions serão executadas.

```bash
az functionapp create --resource-group NetflixCatalogGroup --consumption-plan-location eastus --runtime node --name NetflixCatalogFunctionApp --storage-account netflixcatalogstorage
```

Isso criará um ambiente de funções no qual você pode hospedar e executar funções sem servidor.

### **2. Criando a Azure Function para Salvar Arquivos no Storage Account**

Agora, vamos criar uma **Azure Function** para carregar arquivos para o **Azure Storage Account**. Esse tipo de função pode ser ativado por eventos HTTP.

#### **Passo 1: Criar a Função**

Dentro do seu projeto de Functions, crie uma nova função que aceite arquivos enviados via HTTP. No Visual Studio Code, instale a extensão do **Azure Functions** e crie um novo projeto.

```bash
func init NetflixCatalogFunctionApp --javascript
cd NetflixCatalogFunctionApp
func new
```

Escolha o template **HTTP trigger**.

#### **Passo 2: Código para Salvar Arquivos no Storage Account**

Dentro da função criada, adicione o seguinte código para salvar arquivos no **Azure Blob Storage**:

```javascript
const { BlobServiceClient } = require('@azure/storage-blob');

module.exports = async function (context, req) {
    const connectionString = process.env.AzureWebJobsStorage;
    const containerName = 'movies';
    const blobServiceClient = BlobServiceClient.fromConnectionString(connectionString);
    const containerClient = blobServiceClient.getContainerClient(containerName);

    const fileName = req.body.filename;
    const fileBuffer = req.body.fileBuffer; // Certifique-se de que o arquivo está no corpo da requisição

    const blockBlobClient = containerClient.getBlockBlobClient(fileName);
    await blockBlobClient.upload(fileBuffer, fileBuffer.length);

    context.res = {
        status: 200,
        body: "Arquivo carregado com sucesso"
    };
};
```

Este código salva o arquivo em um contêiner chamado "movies" no seu Storage Account. O arquivo é enviado por meio de uma solicitação HTTP.

#### **Passo 3: Configuração e Deploy**

Você precisará adicionar a chave de conexão para o **Azure Blob Storage** no arquivo `local.settings.json` ou no painel de configurações de aplicação do Azure.

Agora, basta fazer o deploy da sua função:

```bash
func azure functionapp publish NetflixCatalogFunctionApp
```

### **3. Criando a Azure Function para Filtrar Registros no CosmosDB**

Agora, criaremos uma função que filtra os registros do catálogo no **CosmosDB**, buscando informações como nome do filme, gênero, ou qualquer outra propriedade.

#### **Passo 1: Criar a Função de Filtro**

Adicione uma nova função HTTP trigger para filtrar os registros.

```bash
func new
```

Escolha o template **HTTP trigger** e adicione o código para filtrar os registros no CosmosDB. O código pode ser algo assim:

```javascript
const { CosmosClient } = require('@azure/cosmos');

module.exports = async function (context, req) {
    const endpoint = process.env.COSMOSDB_ENDPOINT;
    const key = process.env.COSMOSDB_KEY;
    const databaseId = 'NetflixCatalogDB';
    const containerId = 'Movies';

    const client = new CosmosClient({ endpoint, key });
    const container = client.database(databaseId).container(containerId);

    const query = 'SELECT * FROM c WHERE c.genre = @genre';
    const parameters = [
        { name: '@genre', value: req.query.genre }
    ];

    const { resources: movies } = await container.items.query({ query, parameters }).fetchAll();

    context.res = {
        status: 200,
        body: movies
    };
};
```

Esta função filtra os filmes com base no gênero (passado como parâmetro na URL).

#### **Passo 2: Configuração e Deploy**

Como na função anterior, defina as configurações de **CosmosDB** no `local.settings.json` ou nas configurações da função no portal do Azure.

Faça o deploy novamente:

```bash
func azure functionapp publish NetflixCatalogFunctionApp
```

### **4. Criando a Azure Function para Listar Registros no CosmosDB**

Agora, vamos criar uma função para listar todos os registros armazenados no CosmosDB. Isso seria útil para exibir todos os filmes no catálogo.

#### **Passo 1: Criar a Função de Listagem**

Adicione mais uma função **HTTP trigger** para listar os registros:

```bash
func new
```

Escolha o template **HTTP trigger** e adicione o seguinte código para listar todos os filmes:

```javascript
const { CosmosClient } = require('@azure/cosmos');

module.exports = async function (context, req) {
    const endpoint = process.env.COSMOSDB_ENDPOINT;
    const key = process.env.COSMOSDB_KEY;
    const databaseId = 'NetflixCatalogDB';
    const containerId = 'Movies';

    const client = new CosmosClient({ endpoint, key });
    const container = client.database(databaseId).container(containerId);

    const query = 'SELECT * FROM c';
    const { resources: movies } = await container.items.query(query).fetchAll();

    context.res = {
        status: 200,
        body: movies
    };
};
```

#### **Passo 2: Configuração e Deploy**

Certifique-se de que você tem as credenciais do **CosmosDB** configuradas corretamente e faça o deploy:

```bash
func azure functionapp publish NetflixCatalogFunctionApp
```

### **Conclusão**

Você agora tem um gerenciador de catálogos da Netflix simples com **Azure Functions** e **CosmosDB**. O fluxo geral é:

- **Azure Function para Salvar Arquivos**: Carrega arquivos de mídia (como imagens de filmes) para o Azure Storage.
- **Azure Function para Filtrar Registros no CosmosDB**: Permite consultar os filmes armazenados no CosmosDB com base em um filtro, como o gênero.
- **Azure Function para Listar Registros**: Exibe todos os filmes no banco de dados.

Essas funções podem ser expandidas e aprimoradas, por exemplo, adicionando autenticação, tratamento de erros, validações e mais. Você também pode configurar uma interface de usuário (front-end) para consumir essas APIs e exibir o catálogo de maneira interativa!
