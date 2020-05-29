# Manual Node JS

## Configurações iniciais de um projeto node com typescript

### Primeiros passos

- Ao iniciar um projeto, rodar o comando `yarn init -y`
- Como a maioria dos projetos usará o typescript, é necessário instalar ele com `yarn add typescript`
- Após intalar, gere o arquivo **tsconfig.json** com o comando `yarn tsc --init`. Após fazer isso, crie uma pasta `src/` e altere as seguintes linhas no arquivo:

``` json
{
    ...
    "outDir": "./dist",                        
    "rootDir": "./src",
    ...
}       
```

Isso irá informar ao transpilador typescript que o diretório raiz se chama **src** e o diretório que irá guardar a build do projeto se chama **build**

- Rode `yarn add ts-node-dev -D` para instalar o binário de execução do projeto em desenvolvimento
- Para iniciar a aplicação, digite `yarn dev:server`
- Crie uma nova regra **scripts** no arquivo `package.json`:

```json
{
    "scripts": {
        "build": "tsc",
        "dev:server": "ts-node-dev --inspect --transpile-only --ignore-watch node_modules src/server.ts"
    } 
}
```

### EditorConfig

- Pra configurar o padrão de código para múltiplos editores, crie um arquivo `.editorconfig` e coloque o seguinte:
  
```conf
root = true

[*]
end_of_line = lf
indent_style = space
indent_size = 2
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

```

- Importante mencionar que seu editor deve ter a extensão para entender o arquivo

### Eslint && Prettier

#### Instalação do Eslint

- Iremos adicionar o eslint para padronizar o typescript no projeto: `yarn add eslint -D`. Após isso, rode `yarn eslint --init`

- Responda as perguntas da seguinte maneira:

? How would you like to use ESLint? <span style="color:cyan;">To check syntax, find problems, and enforce code style</span>

? What type of modules does your project use? <span style="color:cyan">JavaScript modules (import/export)</span>

? Which framework does your project use? <span style="color:cyan">None of these</span>

? Does your project use TypeScript? <span style="color:cyan">Yes</span>

? Where does your code run? <span style="color:cyan">Node</span>

? How would you like to define a style for your project? <span style="color:cyan">Use a popular style guide</span>


? Which style guide do you want to follow? <span style="color:cyan">Airbnb: https://github.com/airbnb/javascript</span>

? What format do you want your config file to be in? <span style="color:cyan">JSON</span>

? Would you like to install them now with npm? <span style="color:cyan">no</span>


- Após isso, rodar o seguinte comando:
`yarn add -D @typescript-eslint/eslint-plugin@latest eslint-config-airbnb-base@latest eslint-plugin-import@^2.20.1 @typescript-eslint/parser@latest` , para instalar as dependências com o yarn e não com o npm (motivo para dizer não à última pergunta)

- A extensão do eslint deve estar instalada no seu editor

- Para o caso do VScode, abra as configurações no formato de JSON e coloque o seguinte (para habilitar a correção ao salvar o arquivo):

```json
{
    "[javascript]": {
        "editor.codeActionsOnSave": {
            "source.fixAll.eslint": true
        }
    },

    "[javascriptreact]": {
        "editor.codeActionsOnSave": {
            "source.fixAll.eslint": true
        }
    },

    "[typescript]": {
        "editor.codeActionsOnSave": {
            "source.fixAll.eslint": true
        }
    },

    "[typescriptreact]": {
        "editor.codeActionsOnSave": {
            "source.fixAll.eslint": true
        }
    }
}
```

- Rode `yarn add eslint-import-resolver-typescript -D`

#### Instalação do Prettier

- Rode `yarn add prettier eslint-config-prettier eslint-plugin-prettier -D`
- A extensão do prettier deve estar instalada no seu editor


#### Configurações Gerais

- Um arquivo chamado `.eslintrc.json` foi criado e ele precisa ser configurado. Copie esse arquivo:

```json
{
  "env": {
    "es6": true,
    "node": true
  },
  "extends": [
    "airbnb-base",
    "plugin:@typescript-eslint/recommended",
    "prettier/@typescript-eslint",
    "plugin:prettier/recommended"
  ],
  "globals": {
    "Atomics": "readonly",
    "SharedArrayBuffer": "readonly"
  },
  "parser": "@typescript-eslint/parser",
  "parserOptions": {
    "ecmaVersion": 11,
    "sourceType": "module"
  },
  "plugins": [
    "@typescript-eslint",
    "prettier"
  ],
  "rules": {
    "prettier/prettier": "error",
    "import/extensions": [
      "error",
      "ignorePackages",
      {
        "ts": "never"
      }
    ]
  },

  "settings": {
    "import/resolver": {
      "typescript": {}
    }
  }
}

```

- Crie um arquivo chamado `.eslintignore` para que alguns pacotes sejam ignorados pelo eslint e coloque:
```
/*.js
node_modules
dist
```

- Crie um arquivo chamado `prettier.config.js` para configurações adicionais do prettier:

```javascript
module.exports = {
  singleQuote: true,
  trailingComma: 'all',
  arrowParens: 'avoid',
};
```

### Debugger Node (Apenas para o VSCode)

- Abra um menu de debug e crie um  arquivo `launch.json` (só clicar)
- Substitua o configurations com as configurações abaixo por:
```json
{
    "configurations": [
        {
            "type": "node",
            "request": "attach",
            "protocol": "inspector",
            "restart": true,
            "name": "Debug",
            "skipFiles": [
                "<node_internals>/**"
            ]
        }
    ]
}
```

- Para usar o debugger, basta ir no menu de debug e clicar em RUN