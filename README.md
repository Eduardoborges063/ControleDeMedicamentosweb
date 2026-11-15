# Controle de Medicamentos

Aplicação web em **ASP.NET Core MVC (.NET 8)** para gerenciar o dia a dia de uma clínica ou farmácia: cadastro de funcionários, fornecedores, pacientes e medicamentos, registro de prescrições médicas e controle de estoque por meio de requisições de entrada e saída.

## Funcionalidades

| Módulo | O que faz |
| --- | --- |
| **Funcionários** | Cadastro, edição, exclusão e listagem de funcionários. |
| **Fornecedores** | Cadastro, edição, exclusão e listagem de fornecedores (com CNPJ). |
| **Pacientes** | Cadastro, edição, exclusão e listagem de pacientes (Cartão SUS e CPF). |
| **Medicamentos** | Cadastro de medicamentos vinculados a um fornecedor, com indicação de **medicamento em falta** (estoque abaixo de 20 unidades). |
| **Prescrições** | Criação de prescrições para pacientes e gerenciamento dos medicamentos prescritos. |
| **Estoque (Requisições)** | **Entrada**: adiciona unidades ao estoque. **Saída**: dispensa os medicamentos de uma prescrição, em etapas, validando se há estoque suficiente. |

## Tecnologias

- .NET 8 / ASP.NET Core MVC (Razor Views)
- Bootstrap, jQuery e jQuery Validation
- Dapper + Microsoft.Data.SqlClient (módulo Pacientes, em SQL Server)
- Serilog (console, arquivo e New Relic)
- Persistência em arquivo JSON (demais módulos)

## Estrutura da solução

```
ControleDeMedicamentos.sln
├── ControleDeMedicamentos.Dominio                 # Entidades e regras de negócio
├── ControleDeMedicamentos.Infraestrutura.Arquivos # Repositórios com persistência em dados.json
├── ControleDeMedicamentos.Infraestrutura.SqlServer# Repositórios com Dapper (hoje: Paciente)
├── ControleDeMedicamentos.Util.SqlProject         # Projeto SQL com o script da tabela TBPaciente
└── ControleDeMedicamentos.WebApp                  # Controllers, ViewModels, Views e configuração
```

## Pré-requisitos

- [.NET SDK 8.0](https://dotnet.microsoft.com/download/dotnet/8.0)
- Uma instância do **SQL Server** (LocalDB, Docker ou servidor) para o módulo **Pacientes**
- (Opcional) Visual Studio 2022 ou VS Code

## Como executar

### 1. Clonar e restaurar

```bash
git clone <url-do-repositorio>
cd ControleDeMedicamentosweb-master
dotnet restore
```

### 2. Criar o banco de dados (módulo Pacientes)

Crie um banco (por exemplo, `ControleDeMedicamentos`) e execute o script abaixo, que também está em `ControleDeMedicamentos.Util.SqlProject/TBPaciente.sql`:

```sql
CREATE TABLE [dbo].[TBPaciente]
(
    [Id]        UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
    [Nome]      NVARCHAR(100)    NOT NULL,
    [Telefone]  NVARCHAR(100)    NOT NULL,
    [CartaoSus] NVARCHAR(20)     NOT NULL,
    [Cpf]       NVARCHAR(20)     NOT NULL
);
```

### 3. Configurar as variáveis de configuração

A aplicação lê duas chaves de configuração (variáveis de ambiente, User Secrets ou `appsettings`):

| Chave | Obrigatória | Descrição |
| --- | --- | --- |
| `SQL_CONNECTION_STRING` | Sim, para Pacientes | Connection string do SQL Server. |
| `NEWRELIC_LICENSE_KEY` | Não* | Chave de licença do New Relic para envio de logs. |

\* O Serilog está configurado para enviar logs ao New Relic. Se você não tem uma chave, defina qualquer valor ou remova o sink `NewRelicLogs` em `DependencyInjection/SerilogConfig.cs`.

Usando **User Secrets** (recomendado em desenvolvimento; o projeto já possui `UserSecretsId`):

```bash
cd ControleDeMedicamentos.WebApp
dotnet user-secrets set "SQL_CONNECTION_STRING" "Server=localhost;Database=ControleDeMedicamentos;User Id=sa;Password=SuaSenha;TrustServerCertificate=True"
dotnet user-secrets set "NEWRELIC_LICENSE_KEY" "sua-chave-ou-valor-qualquer"
```

Ou, por variável de ambiente:

```bash
export SQL_CONNECTION_STRING="Server=localhost;Database=ControleDeMedicamentos;User Id=sa;Password=SuaSenha;TrustServerCertificate=True"
export NEWRELIC_LICENSE_KEY="sua-chave-ou-valor-qualquer"
```

> Dica: para subir o SQL Server rapidamente com Docker:
> `docker run -e ACCEPT_EULA=Y -e MSSQL_SA_PASSWORD='SuaSenha@123' -p 1433:1433 -d mcr.microsoft.com/mssql/server:2022-latest`

### 4. Executar

```bash
cd ControleDeMedicamentos.WebApp
dotnet run --launch-profile http
```

Acesse **http://localhost:5197**.

Perfis disponíveis em `launchSettings.json`:

- `http` → `http://localhost:5197`
- `https [Dev]` → `https://localhost:7001` e `http://localhost:5197`
- `https [Prod]` → mesmo endereço, com `ASPNETCORE_ENVIRONMENT=Production`

## Armazenamento de dados

- **Pacientes** são gravados no **SQL Server** (`RepositorioPacienteEmSql`, via Dapper).
- **Funcionários, fornecedores, medicamentos, prescrições e requisições** são gravados em um arquivo JSON (`dados.json`) na pasta `ApplicationData` do usuário:
  - Windows: `%APPDATA%\ControleDeMedicamentos\dados.json`
  - Linux: `~/.config/ControleDeMedicamentos/dados.json`
  - macOS: `~/Library/Application Support/ControleDeMedicamentos/dados.json`
- Logs de nível **Error** também são gravados em `erro.log` na mesma pasta.

## Regras de negócio principais

- O estoque de um medicamento é a soma das requisições de **entrada** menos as quantidades das prescrições dispensadas nas requisições de **saída**.
- Um medicamento é considerado **em falta** quando o estoque é menor que 20 unidades.
- Uma requisição de saída só é aceita se houver estoque suficiente para cada medicamento da prescrição.

## Problemas conhecidos

- O botão **Estoque** da página inicial aponta para um controller `Estoque` que não existe (o correto é `RequisicaoMedicamento`, como no menu do layout), então esse link retorna 404.
- A tela de **Pacientes** depende do SQL Server e falha se `SQL_CONNECTION_STRING` não estiver configurada ou se a tabela não existir.
- `PacienteController` não trata o caso de paciente não encontrado nas ações `Editar` e `Excluir` (pode gerar erro de referência nula).
- Existem `RepositorioPacienteEmArquivo` e `RepositorioPacienteEmSql` ao mesmo tempo; a migração dos demais módulos para SQL Server ainda não foi feita.

## Licença

Defina aqui a licença do projeto.
