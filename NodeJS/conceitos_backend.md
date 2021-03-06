# Conceitos Backend NodeJS

## Conceitos básicos

### Models
Funcionam do mesmo jeito que no rails, são classes que armazenam o formato dos dados de uma determinada entidade

### Repository
Os repositórios são a interface de manipulação da persistência das models no sistema. Eles devem armazenas o código de CRUD e funções que precisam ler e manipular dados

### Services
Um service é uma ação na aplicação. Cada service é uma classe e deve conter apenas uma função `public execute()` ou `public run()`. Esse serviço deve conter a regra de negócio da aplicação

### Rotas
No express cada rota é um parametro para uma certa função. Dentro de uma pasta `routes`, crie um objeto router para usar como sufixo de todas as rotas que fazem sentido juntas. Em outro arquivo chamado `<nome>.routes.ts` coloque todas as rotas adequadas e exporte o router. Dentro de um arquivo index, faça o seguinte:

```ts
import { Router } from 'express';
import appointmentsRouter from './appointments.routes';

const routes = Router();

routes.use('/appointments', appointmentsRouter);
```

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

### Relacionamentos entre as Models

#### OneToMany ou ManyToOne
A relação 1-N pode ser chamada de OneToMany ou ManyToOne dependendo do referencial. Mas basicamente ela trata o caso de uma coluna de uma tabela fazer referência a outra tabela. Isso é feito pelo uso de chaves estrangeiras (FKs)

No TypeOrm, a seguinte migração deve ser feita no método `up`:

```ts
await queryRunner.createForeignKey(
      'appointments',
      new TableForeignKey({
        name: 'appointment_provider',
        columnNames: ['provider_id'],
        referencedColumnNames: ['id'],
        referencedTableName: 'users',
        onDelete: 'SET NULL',
        onUpdate: 'CASCADE',
      }),
    );
```
Onde

- 'appointments' é a tabela de origem que vai referenciar outra tabela
- **name** é o apelido da FK
- **columnNames** é o nome do campo na tabela de origem
- **referencedColumnNames** é o nome do campo na tabela de destino
- **referencedTableName** é o nome da tabela de destino
- **onDelete** diz o que fazer com as referências quando o campo de origem for deletado
- **onUpdate** diz o que fazer com as referências quando o campo de origem for atualizado

No método `down`:

```ts
    await queryRunner.dropForeignKey('appointments', 'appointment_provider');
```

Tenha em mente que
- os tipos no banco devem ser iguais
- a tabela de destino já deve estar criada

Já nas models, é necessário adicionar o seguinte:

```ts
@Column()
provider_id: string;

@ManyToOne(() => User)
@JoinColumn({ name: 'provider_id' })
provider: User;
```

Isso faz com que a classe tenha acesso à referência. É possível utilizar o ManyToOne ou o OneToMany, sempre dependendo da referência de onde se olha.

## Autenticação

### Cadastro de usuários
A primeira coisa a se pensar na autenticação é como salvar os dados de um usuário de maneira segura. Para isso, iremos criptografar a senha do usuário. Para isso usaremos o módulo `bcryptjs`, lembrando de instalar também os seu arquivo de tipos como dependência de desenvolvimento

`yarn add bcryptjs` e `yarn add @types/bcryptjs -D`

após isso, no método de salvar o usuário na base de dados:

```ts
import { hash } from 'bcryptjs';
const hashedPassword = await hash(password, 8);

const user = userRepository.create({
  name,
  email,
  password: hashedPassword,
});
```

### Validando credenciais

#### Comparando senhas
Para validar as credenciais de um usuário, basta usar o método `compare` do módulo bcryptjs

```ts
const passwordMatched: bool = await compare(password, user.password);
```

Importante mencionar para sempre retornar mensagens de erro genéricas na autenticação, para evitar que o usuário saiba qual parte ele errou. Dessa maneira, podemos evitar potenciais invasores.

#### Gerando JWTs
Uma das maneiras de autenticar usuários em apis REST é usando JWTs. Usando o módulo `jsonwebtoken` e sua declaração de tipos, podemos assinar um payload (**Jamais vazar a chave secreta**) e devolver na nossa resposta

### Autenticando rotas
Algumas rotas da aplicação vão precisar de autenticação. Faremos isso por meio de um middleware do express. O token será passado num Auth Header do tipo bearer e precisará estar válido em todas as requisições

Dentro de uma pasta `src/middlewares`, iremos criar o nosso middleware de autenticação. Ele irá receber o token e performar as devidas validações

```ts
import { Request, Response, NextFunction } from 'express';
import { verify } from 'jsonwebtoken';

export default function ensureAuthenticated(
  req: Request,
  res: Response,
  next: NextFunction,
): void {
  const authHeader = req.headers.authorization;

  if (!authHeader) {
    throw new Error('JWT token is missing');
  }

  const token = authHeader.split(' ')[1];
  const { secret } = authConfig.jwt;

  try {
    const decoded = verify(token, secret);

    const { sub } = decoded as TokenPayload;

    req.user = {
      id: sub,
    };

    return next();
  } catch {
    throw new Error('Invalid JWT token');
  }
}
```

Repare que a variável decoded está sendo forçada a ter o tipo `TokenPayload`. Isso se dá pelo fato de o payload de um jwt ser variável. Então, para contornar isso, declaramos uma interface com todas as propriedades que precisamos e utilizamos como no código acima

Outro detalhe é que estamos colocando o `sub` dentro da propriedade `id` de um objeto user. Originalmente, o req do express não possui essa propriedade, então devemos usar uma manha do typescript para permitir isso.

Dentro de uma pasta `src/@types`, devemos criar um arquivo `express.d.ts` e preencher da seguinte forma:

```ts
declare namespace Express {
  export interface Request {
    user: {
      id: string;
    };
  }
}
```

Isso vai **adicionar** a propriedade user à interface Request. Isso é interessante pelo fato de podermos tornar a informação que foi calculada no middlware disponível na rota subsequente

Para usarmos esse middleware em alguma rota fazemos:

```ts
import ensureAuthenticated from '../middlewares/ensureAuthenticated';

const appointmentsRouter = Router();

appointmentsRouter.use(ensureAuthenticated);
```