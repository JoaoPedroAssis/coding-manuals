# Conceitos Backend NodeJS

## Conceitos básicos

### Models
Funcionam do mesmo jeito que no rails, são classes que armazenam o formato dos dados de uma determinada entidade

### Repository
Os repositórios são a interface de manipulação da persistência das models no sistema. Eles devem armazenas o código de CRUD e funções que precisam ler e manipular dados

### Services
Um service é uma ação na aplicação. Cada service é uma classe e deve conter apenas uma função `public execute()` ou `public run()`. Esse serviço deve conter a regra de negócio da aplicação

## Configuração de BD com TypeOrm

Uma maneira massa de ter um bd rodando sem ficar poluindo seu pc com um monte de coisa é usar o docker. Tendo o docker instalado rode:

`docker run --name <nome do container> -e POSTGRES_PASSWORD=<senha> -p 5432:5432 -d <imagem>`

Onde:
- name: Nome do container na sua maquina
- senha: senha do root do BD
- -p: porta que o docker vai usar, no formato <porta_do_docker>:<porta_do_BD>
- imagem: imagem do docker hub, nesse caso fica a do postgres

ex:

`docker run --name gostack_postgres -e POSTGRES_PASSWORD=gostack -p 5432:5432 -d postgres`

Instale o typeorm, o driver do postgres (ou do BD que vc escolher) e o pacote reflect-metadata, para podermos usar os decorators (veja mais abaixo)

`yarn add typeorm pg reflect-metadata`


e crie um arquivo `ormconfig.json` na raiz, com o seguinte conteúdo:

```json
{
  "type": "postgres",
  "host": "localhost",
  "port": "5432",
  "username": "<username>",
  "password": "<senha>",
  "database": "<database>",
  "entities": [
    "./src/models/*.ts"
  ],
  "migrations": [
    "./src/database/migrations/*.ts"
  ],
  "cli": {
    "migrationsDir": "./src/database/migrations"
  }
}
```

A database precisa estar previamente criada, então dê um jeito de cria ela dentro do SGBD (usando o DataGrip por exemplo)

Depois disso, crie uma pasta chamada `database/` com um arquivo index.ts e coloque:

```typescript
import { createConnection } from 'typeorm';

createConnection();
```

agora importe a conexão e o reflect-metadata no server.ts (arquivo principal da aplicação). O módulo reflect-metadata deve ser o primeiro:

```typescript
import 'reflect-metadata';
...
import './database';
```

e adicione 

`"typeorm": "ts-node-dev ./node_modules/typeorm/cli.js"` na aba scripts do seu package.json

## Criando Migrações

### Criando a migração
Para criar uma migração, rode o comando
`yarn typeorm migration:create -n <nomeDaMigração>`, que irá criar uma nova migração em `database/migrations`, que tem essa cara aqui:

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class CreateAppointments1590108921819
  implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {}

  public async down(queryRunner: QueryRunner): Promise<void> {}
}
```
- Pro eslint parar de chorar, coloque um `export default` e faça o retorno das Promises serem void ao invés de any (caso apareça assim)

O método `up` irá ser executando quando a migração for rodada e o `down` quando ela for revertida, então é muito importante que este desfaça o feito por aquele

- pra rodar o up: `yarn typeorm migration:run`
- pra rodar o down: `yarn typeorm migration:revert`

### Criando a tabela
Dentro de qualquer uma das funções, deve se utilizar a interface do queryRunner para fazer qualquer tipo de ação na tabela. Segue um exemplo:

```typescript
await queryRunner.createTable(
      new Table({
        name: 'appointments',
        columns: [
          {
            name: 'id',
            type: 'varchar',
            isPrimary: true,
            generationStrategy: 'uuid',
            default: 'uuid_generate_v4()'
          },
          {
            name: 'provider',
            type: 'varchar',
            isNullable: false,
          },
          {
            name: 'date',
            type: 'timestamp with time zone',
            isNullable: false,
          },
        ],
      }),
    );
```

Ler a docs é importante e algumas coisas podem varia dependendo do driver utilizado

## Models

### Declarando as Models
As models no typeorm vão precisar de configurações especiais para se conectarem a uma tabela. Elas ainda são classes, mas usam os Decorators do typescript. Então no arquivo de models faça o seguinte:

`import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';`
  
Essa imaportação puxa os decorators de entidade, coluna e coluna com valor gerado automaticamente. 
- Entity: linka a classe à tabela
- Column: linka a propriedade à coluna da tabela (podemos ter propriedades que não são colunas)
- PrimaryGeneratedColumn: Coluna de chave primária com inicialização automática

```typescript
@Entity('appointments')
export default class Appointment {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  provider: string;

  @Column('time with time zone')
  date: Date;
}
```

É impostante fazer as seguintes alterações no tsconfig.json para que os decorators e o typeorm funcionem corretamente:

```json
{
  "strictPropertyInitialization": false,
  "experimentalDecorators": true,
  "emitDecoratorMetadata": true
}
```
É sempre massa ter colunas pra gravar quando o asset foi criado e quando o asset foi atualizado. Pra fazer isso, basta adicionar os seguintes campos na model:

```ts
@CreateDateColumn()
created_at: Date;

@UpdateDateColumn()
updated_at: Date;
```

E na migração:

```ts
{
  name: 'created_at',
  type: 'timestamp',
  default: 'now()',
},
{
  name: 'updated_at',
  type: 'timestamp',
  default: 'now()',
},
```

### Criando as Models com os repositórios e services

O typeorm fornece um repositório padrão, com boa parte dos métodos necessários (como se fosse o ActiveRecord). Então precisamos fazer com que o nosso repositório tenha acesso a esses métodos:

```ts
import { EntityRepository, Repository } from 'typeorm';

@EntityRepository(Model)
export default class AppointmentsRepository extends Repository<Model>
```
A maioria dos métodos de acesso ao bd são assíncronos, então ficar esperto e colocar `async/await` onde deve e lembrar de colocar o retorno como uma Promise.

Para fazer a criação ou alteração, os métodos fornecidos retornam uma instância do objeto criado/alterad. Esses métodos não são assíncronos e precisarão que a função de `save()` seja chamada:

```ts
const appointment = appointmentsRepository.create({
  provider,
  date: appointmentDate,
});

await appointmentsRepository.save(appointment);

return appointment;
```