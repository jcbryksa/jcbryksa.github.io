## API REST con Loopback 4

**Loopback 4** es un framework para la creación de APIs en [Node.js](https://nodejs.org/) basado en [Express](https://expressjs.com/). Permite el desarrollo tanto en lenguaje JavaScript como en Typescript.
A continuación, se describe paso a paso la construcción de una API REST con esta tecnología, tomando como base los instructivos presentados en la <a href="https://loopback.io/doc/en/lb4/index.html" target="_blank">documentación oficial</a>.


> __Requisitos previos__:

>* [Instalación de Node.js](https://nodejs.org/en/download/)


---

## Creación del proyecto

En caso que aún no esté, por única vez se deberá instalar LoopBack 4 CLI de forma global:

```
npm i -g @loopback/cli
```

&nbsp;
Luego la creación de una nueva aplicación:

```
lb4 app
```

&nbsp;
El comando va pidiendo completar los parámetros de manera interactiva. Se ignoran las siguientes caracteristicas: docker, eslint, prettier y mocha.

```
? Nombre de proyectos: catalog-manager-lb-api
? Descripción de proyecto: Catalog Manager API
? Directorio raíz de proyecto: catalog-manager-lb-api
? Nombre de clase de aplicación: CatalogManagerLbApiApplication
? Seleccionar características para habilitarlas en el proyecto
 (*) Enable loopbackBuild: utilizar ayudantes @loopback/build (p. ej. lb-eslint)
 (*) Enable vscode: añadir archivos de configuración VSCode
 ( ) Enable docker: incluir archivo Docker y .dockerignore
>(*) Enable repositories: incluir importaciones de repositorio y RepositoryMixin
 (*) Enable services: incluir importaciones de proxy de servicio y ServiceMixin
 ( ) Enable eslint: añadir linter con reglas de lint configuradas previamente
 ( ) Enable prettier: instalar prettier para formatear código según las reglas
(Move up and down to reveal more choices)
```

&nbsp;
Luego se ejecuta la aplicación para verificar que todo se haya creado correctamente.

```bash
cd catalog-manager-lb-api
npm start
```

&nbsp;
Y se accede desde el navegador al host local mediante el puerto 3000 que se configura por defecto (http://localhost:3000/explorer/), visualizando la interfaz de Swagger:

<img src="assets/swagger-loopback-01.png" />

---

## Orígenes de Datos

Se creará un origen para la conexión con la base de datos **Sql Server** existente, ejecutando el siguiente comando:

```
lb4 datasource
```

&nbsp;
El comando va pidiendo completar los parámetros de manera interactiva:

```
? Nombre de Datasource: mssqlCatalogManager
? Select the connector for mssqlCatalogManager: Microsoft SQL (supported by StrongLoop)
? Connection String url to override other settings (eg: mssql://username:password@localhost/database): mssql://srv_catalog_manager_user:123456@jcb-lenovo:1433/CatalogMa
nager
? host: jcb-lenovo
? port: 1433
? user: srv_catalog_manager_user
? password: [hidden]
? database: CatalogManager
```

---

## Modelos y Repositorios

Loopback maneja los conceptos `Model` y `Repository`, siendo el primero de ellos el que define la estructura de los datos y el segundo quien establece las operaciones realizadas con los mismos (CRUD).

Se procederá con la **creación de los modelos importados desde la base de datos** mediante el siguiente comando:

```
lb4 discover
```

Warning: **Importante.** Antes de ejecutar esta acción se debió generar un build con el origen datos creado. Caso contrario dicho origen no será reconocido. <br><br> <span style="color:red;">No esta soportado el tipo BINARY de SQL Server.</span>

&nbsp;
De manera interactiva se le indicará que utilice el origen de datos creado anteriormente y se seleccionarán las tablas (todas) para las cuales se desea la creación de modelos:

```
? Seleccione el conector que se va a descubrir mssqlCatalogManager
? Seleccione los modelos a descubrir
 (*) users
 (*) user_roles
 (*) user_permissions
>(*) catalogs
 (*) layouts
 (*) categories
 (*) attribute_types
(Move up and down to reveal more choices)
```

&nbsp;
Luego se procede a generar los repositorios correspondientes:

```
lb4 repository
```

&nbsp;
De manera interactiva se le indicará que utilice el origen de datos creado anteriormente, los modelos (todoos) y la clase base para los repositorios, que será la que trae el framework por defecto:

```
? Seleccione el origen de datos MssqlCatalogManagerDatasource
? Seleccione el modelo o los modelos que desea para generar un repositorio AttributeTypes, Attributes, Catalogs, Categories, CategoryRelations, ItemCategories, ItemRela
tions, ItemTypeRelations, ItemTypes, Items, Layouts, Permissions, RolePermissions, Roles, UserPermissions, UserRoles, Users
? Seleccione la clase base del repositorio DefaultCrudRepository (Legacy juggler bridge)
```

---

## Controladores

Se creará cada controlador, pudiendose especificar `Model` y `Repository` para crear de forma automática todos los métodos.

```
lb4 controller
```

&nbsp;
De manera interactiva se le indicará el nombre, modelo, repositorio, id único, tipo de dato para el id y ruta:

```
? Nombre de clase Controller: Users
Controller Users will be created in src/controllers/users.controller.ts

? ¿Qué especie de controlador desea generar? REST Controller with CRUD functions
? ¿Cuál es el nombre del modelo que se va a utilizar con este repositorio CRUD? Users
? ¿Cuál es el nombre del repositorio CRUD? UsersRepository
? ¿Cuál es el nombre de la propiedad de ID? id
? ¿Cuál es el tipo del ID? number
? ¿Se omite el ID cuando se crea una instancia nueva? Yes
? ¿Cuál es el nombre de la vía de acceso HTTP base de las operaciones CRUD? /users
```

&nbsp;
Se visualizan los metodos creados en la interfaz de Swagger:

<img src="assets/swagger-users-controller.png" />


---

## Autenticación JWT

Se incorporará la autenticación por JSON Web Tokens (JWT), basado en un <a href="https://github.com/strongloop/loopback-next/tree/master/examples/todo-jwt" target="_blank">ejemplo</a>.

&nbsp;
Primero se agregan las siguientes dependencias en **package.json**, ejecutando luego el comando `npm install`:

```
. . .
"@loopback/authentication": "^4.2.8",
"@loopback/authentication-jwt": "^0.4.2",
"@loopback/security": "^0.2.13",
. . .
```

&nbsp;
En el archivo **sequence.ts**:

&nbsp;
Se agregan los siguientes imports:
```javascript
// Autenticacion JWT
import {
  AuthenticateFn,
  AuthenticationBindings,
  AUTHENTICATION_STRATEGY_NOT_FOUND,
  USER_PROFILE_NOT_FOUND
} from '@loopback/authentication';
//
```

&nbsp;
Se inyecta el método `authenticateRequest` y se agrega la invocación al mismo como middleware:
```javascript
  constructor(
    . . .
    @inject(AuthenticationBindings.AUTH_ACTION)
    protected authenticateRequest: AuthenticateFn,
  ) {}

  async handle(context: RequestContext) {
    try {

      const {request, response} = context;

      const route = this.findRoute(request);

      await this.authenticateRequest(request);

      const args = await this.parseParams(request, route);
      const result = await this.invoke(route, args);
      this.send(response, result);

    } catch (error) {
      // if error is coming from the JWT authentication extension
      // make the statusCode 401
      if (
        error.code === AUTHENTICATION_STRATEGY_NOT_FOUND ||
        error.code === USER_PROFILE_NOT_FOUND
      ) {
        Object.assign(error, {statusCode: 401 /* Unauthorized */});
      }

      this.reject(context, error);
    }
  }
```

&nbsp;
En el archivo **application.ts** se agregan los siguientes imports:

```
// Autenticacion JWT
import {AuthenticationComponent} from '@loopback/authentication';
import {
  JWTAuthenticationComponent,
} from '@loopback/authentication-jwt';
//
```

&nbsp;
para agregar los componentes `AuthenticationComponent` y `JWTAuthenticationComponent`:

```
export class CatalogManagerLbApiApplication extends BootMixin(
  ServiceMixin(RepositoryMixin(RestApplication)),
) {
  constructor(options: ApplicationConfig = {}) {
    super(options);

    // Set up the custom sequence
    this.sequence(MySequence);

    // Set up default home page
    this.static('/', path.join(__dirname, '../public'));

    // Customize @loopback/rest-explorer configuration here
    this.configure(RestExplorerBindings.COMPONENT).to({
      path: '/explorer',
    });
    this.component(RestExplorerComponent);

    this.projectRoot = __dirname;
    // Customize @loopback/boot Booter Conventions here
    this.bootOptions = {
      controllers: {
        // Customize ControllerBooter Conventions here
        dirs: ['controllers'],
        extensions: ['.controller.js'],
        nested: true,
      },
    };

    // ------ ADD SNIPPET AT THE BOTTOM ---------
    // Mount authentication system
    this.component(AuthenticationComponent);
    // Mount jwt component
    this.component(JWTAuthenticationComponent);
    // ------------- END OF SNIPPET -------------

  }
}
```


&nbsp;
Luego en el controlador se agregan los siguientes imports:

```
// Autenticacion JWT
import {authenticate, TokenService} from '@loopback/authentication';
import {
  Credentials,
  TokenServiceBindings,
} from '@loopback/authentication-jwt';
import {inject} from '@loopback/core';
import * as crypto from 'crypto';
//
```

&nbsp;
A continuación se describen los métodos *login()* y *me()* donde se puede observar la forma en que se establece el token y luego la forma de utilizarlo con la anotación **@authenticate('jwt')**:

```
const CredentialsSchema = {
  type: 'object',
  required: ['email', 'password'],
  properties: {
    email: {
      type: 'string',
      format: 'email',
    },
    password: {
      type: 'string',
      minLength: 8,
    },
  },
  example: {
    email: 'jcbryksa@gmail.com',
    password: 'password001'
  }
};

export const CredentialsRequestBody = {
  description: 'The input of login function',
  required: true,
  content: {
    'application/json': {schema: CredentialsSchema},
  },
};


export class UsersController {
  constructor(
    @repository(UsersRepository)
    public usersRepository: UsersRepository,
    @inject(TokenServiceBindings.TOKEN_SERVICE)
    public jwtService: TokenService,
    @inject(SecurityBindings.USER, {optional: true})
    public user: UserProfile,
  ) {}

  @post('/users/login', {
    responses: {
      '200': {
        description: 'Token',
        content: {
          'application/json': {
            schema: {
              type: 'object',
              properties: {
                token: {
                  type: 'string',
                },
              },
            },
          },
        },
      },
    },
  })
  async login(
    @requestBody(CredentialsRequestBody) credentials: Credentials,
  ): Promise<{token: string}> {

    const md5 = (contents: string) => crypto.createHash('md5').update(contents).digest("hex");

    // Busca usuario valido
    let filter = {where: {email: credentials.email, password: md5(credentials.password)}};
    let foundUser = await this.usersRepository.findOne(filter);
    if (!foundUser) {
      throw new HttpErrors.Unauthorized("Error de autenticacion");
    }
    //

    // Crea un JSON Web Token basado en UserProfile
    const userProfile: UserProfile = {
      [securityId]: foundUser.id.toString(),
      name: foundUser.name,
      id: foundUser.id,
      email: foundUser.email
    };
    const token = await this.jwtService.generateToken(userProfile);
    //

    return {token};

  }


  @authenticate('jwt')
  @get('/me', {
    responses: {
      '200': {
        description: 'Usuario autenticado',
        schema: {
          type: 'string',
        },
      },
    },
  })
  async me(
    @inject(SecurityBindings.USER)
    currentUserProfile: UserProfile,
  ): Promise<UserProfile> {
    return currentUserProfile;
  }

 . . .

}
```

&nbsp;
Una vez obtenido el token se lo debe ingresar mediante el botón **Authorize**:

<img src="assets/swagger-lb-auth.png" />

Note: **Importante.** NO se debe anteponer el prefijo "Bearer" al token
