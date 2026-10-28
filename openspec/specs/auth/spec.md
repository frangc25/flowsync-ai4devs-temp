# auth Specification

## Purpose

Cuentas y acceso de FlowSync: una persona puede crear una cuenta, iniciar sesión con email y contraseña, mantener la sesión entre recargas, consultar su perfil y cerrar sesión. La API emite tokens de acceso opacos y la aplicación web protege sus pantallas según haya o no sesión.

## Requirements

### Requirement: Registro de cuenta

La API SHALL permitir crear una cuenta mediante `POST /api/v1/auth/signup` con `fullName`, `email`, `password` y `passwordConfirmation`, y SHALL responder con estado 200 y un cuerpo `{ data: { user, token } }`, donde `user` es el perfil público de la cuenta recién creada y `token` es un token de acceso ya válido para las rutas protegidas.

#### Scenario: Registro correcto con nombre

- **WHEN** se envía un registro con `fullName` "Ada Lovelace", un email no registrado, y `password` y `passwordConfirmation` iguales de entre 8 y 32 caracteres
- **THEN** la respuesta tiene estado 200 y `data.user` contiene el `fullName` y el `email` enviados
- **AND** `data.token` es una cadena que, enviada como `Authorization: Bearer <token>`, da acceso al perfil

#### Scenario: Registro correcto sin nombre

- **WHEN** se envía un registro válido con `fullName: null` o `fullName: ""`
- **THEN** la respuesta tiene estado 200 y `data.user.fullName` es `null`

### Requirement: Validación del registro

La API SHALL rechazar un registro inválido, sin crear la cuenta, con estado 422 y `{ errors: [...] }`, donde cada error indica `field`, `rule`, `message` y, si la regla tiene parámetros, `meta`. `fullName` MUST venir en el cuerpo (puede ser `null`); `email` MUST ser válido, de hasta 254 caracteres y no registrado; ambas contraseñas MUST tener de 8 a 32 caracteres y coincidir.

#### Scenario: Email ya registrado

- **WHEN** se envía un registro con un email que ya pertenece a una cuenta, escrito exactamente igual
- **THEN** la respuesta tiene estado 422 y contiene un error con `field: "email"` y `rule: "database.unique"`

#### Scenario: Datos mal formados

- **WHEN** se envía un registro con email "nope" y contraseña "123"
- **THEN** la respuesta tiene estado 422 y contiene un error `email` sobre `email` y un error `minLength` con `meta.min: 8` sobre `password`

#### Scenario: Contraseña demasiado larga

- **WHEN** se envía un registro con una contraseña de 33 caracteres
- **THEN** la respuesta tiene estado 422 y contiene un error `maxLength` con `meta.max: 32` sobre `password`

#### Scenario: Confirmación distinta

- **WHEN** se envía un registro con `password` y `passwordConfirmation` válidas pero distintas
- **THEN** la respuesta tiene estado 422 y contiene un error con `field: "passwordConfirmation"` y `rule: "sameAs"`

#### Scenario: Falta la clave fullName

- **WHEN** se envía un registro sin la clave `fullName` en el cuerpo
- **THEN** la respuesta tiene estado 422 y contiene un error `required` sobre `fullName`

### Requirement: Emails distintos según mayúsculas

La API SHALL tratar el email de forma literal, distinguiendo mayúsculas y minúsculas, tanto al comprobar si ya está registrado como al iniciar sesión.

#### Scenario: Mismo email con otras mayúsculas

- **WHEN** existe una cuenta con "ana@x.io" y se registra "ANA@X.IO"
- **THEN** la respuesta tiene estado 200 y se crea una segunda cuenta independiente

#### Scenario: Login con otras mayúsculas

- **WHEN** solo existe una cuenta con "ana@x.io" y se inicia sesión con "ANA@X.IO" y la contraseña correcta
- **THEN** la respuesta tiene estado 400 por credenciales inválidas

### Requirement: Inicio de sesión

La API SHALL permitir iniciar sesión mediante `POST /api/v1/auth/login` con `email` y `password`, y ante credenciales correctas SHALL responder con estado 200 y `{ data: { user, token } }`, emitiendo un token nuevo en cada inicio de sesión sin invalidar los anteriores.

#### Scenario: Credenciales correctas

- **WHEN** se inicia sesión con el email y la contraseña de una cuenta existente
- **THEN** la respuesta tiene estado 200, `data.user` es el perfil de esa cuenta y `data.token` da acceso a las rutas protegidas

#### Scenario: Varias sesiones simultáneas

- **WHEN** la misma cuenta inicia sesión dos veces
- **THEN** los dos tokens recibidos son distintos y ambos dan acceso al perfil

### Requirement: Rechazo de credenciales inválidas

La API SHALL responder con estado 400 y `{ errors: [{ message: "Invalid user credentials" }] }` cuando el email no pertenece a ninguna cuenta o la contraseña no es la correcta, sin distinguir entre ambos casos. Si el cuerpo no supera la validación (email con formato inválido o contraseña vacía o ausente), SHALL responder con estado 422 y los errores por campo.

#### Scenario: Contraseña incorrecta

- **WHEN** se inicia sesión con un email existente y una contraseña incorrecta
- **THEN** la respuesta tiene estado 400 y el mensaje "Invalid user credentials"

#### Scenario: Email inexistente

- **WHEN** se inicia sesión con un email que no pertenece a ninguna cuenta
- **THEN** la respuesta es idéntica a la de contraseña incorrecta: estado 400 y el mensaje "Invalid user credentials"

#### Scenario: Cuerpo inválido

- **WHEN** se inicia sesión con email "nope" y contraseña vacía
- **THEN** la respuesta tiene estado 422 con un error `email` sobre `email` y un error `required` sobre `password`

### Requirement: Acceso autenticado por token

Las rutas bajo `/api/v1/account/` SHALL exigir una cabecera `Authorization: Bearer <token>` con un token vigente, y SHALL responder con estado 401 y `{ errors: [{ message: "Unauthorized access" }] }` cuando falte, sea desconocido o haya sido revocado. Los tokens no caducan por tiempo: solo dejan de valer al revocarse. Todas las respuestas de la API SHALL ser JSON independientemente de la cabecera `Accept` de la petición.

#### Scenario: Sin token

- **WHEN** se pide `GET /api/v1/account/profile` sin cabecera `Authorization`
- **THEN** la respuesta tiene estado 401 y el mensaje "Unauthorized access"

#### Scenario: Token inventado

- **WHEN** se pide el perfil con `Authorization: Bearer` seguido de un valor que la API no emitió
- **THEN** la respuesta tiene estado 401

#### Scenario: Cliente que pide HTML

- **WHEN** se pide el perfil sin token y con `Accept: text/html`
- **THEN** la respuesta sigue siendo JSON con estado 401

### Requirement: Consulta del perfil

La API SHALL devolver, en `GET /api/v1/account/profile` con un token vigente, estado 200 y `{ data: user }`, donde `user` contiene exactamente `id`, `fullName`, `email`, `initials`, `createdAt` y `updatedAt` de la cuenta dueña del token, y nunca la contraseña. Estos mismos campos SHALL ser los del `user` devuelto por registro e inicio de sesión.

#### Scenario: Perfil propio

- **WHEN** se pide el perfil con el token obtenido al registrarse
- **THEN** la respuesta tiene estado 200 y `data` contiene los seis campos de la cuenta y ningún campo de contraseña

### Requirement: Iniciales del usuario

La API SHALL calcular `initials` en mayúsculas troceando `fullName` por cada espacio (o, sin `fullName`, el email por la `@`): si los dos primeros trozos no están vacíos, con la primera letra de cada uno; si no, con hasta dos primeras letras del primer trozo, que puede quedar vacío.

#### Scenario: Nombre de varias palabras

- **WHEN** la cuenta tiene `fullName` "Ada Lovelace King"
- **THEN** `initials` es "AL"

#### Scenario: Nombre de una palabra

- **WHEN** la cuenta tiene `fullName` "Ada"
- **THEN** `initials` es "AD"

#### Scenario: Sin nombre

- **WHEN** la cuenta no tiene `fullName` y su email es "spec@x.io"
- **THEN** `initials` es "SX"

#### Scenario: Espacios repetidos o iniciales

- **WHEN** la cuenta se registra en la API con `fullName` "Ada  Lovelace" (dos espacios), o con " Ada"
- **THEN** `initials` es "AD" en el primer caso y la cadena vacía en el segundo

### Requirement: Cierre de sesión en la API

La API SHALL revocar, en `POST /api/v1/account/logout`, únicamente el token con el que se hace la petición y SHALL responder con estado 200 y `{ message: "Logged out successfully" }`, sin el envoltorio `data`.

#### Scenario: Logout correcto

- **WHEN** se hace logout con un token vigente
- **THEN** la respuesta tiene estado 200 con `message: "Logged out successfully"`
- **AND** una petición posterior al perfil con ese mismo token recibe 401

#### Scenario: Logout repetido

- **WHEN** se repite el logout con un token ya revocado
- **THEN** la respuesta tiene estado 401

#### Scenario: Otras sesiones intactas

- **WHEN** una cuenta tiene dos tokens y hace logout con uno de ellos
- **THEN** el otro token sigue dando acceso al perfil

### Requirement: Pantalla de inicio de sesión

La aplicación web SHALL ofrecer en `/login` un formulario "Inicia sesión" con los campos Email y Contraseña, un botón "Entrar" y un enlace "Crea una" que lleva al registro. Al enviarlo con credenciales correctas SHALL abrir la sesión y llevar a la persona a su perfil.

#### Scenario: Login correcto

- **WHEN** una persona sin sesión introduce credenciales correctas en `/login` y pulsa "Entrar"
- **THEN** pasa a ver su pantalla de perfil

#### Scenario: Envío en curso

- **WHEN** la persona pulsa "Entrar" y la respuesta aún no ha llegado
- **THEN** el botón muestra "Entrando…" y está deshabilitado

#### Scenario: Credenciales incorrectas

- **WHEN** la persona envía un email o una contraseña incorrectos
- **THEN** sigue en `/login` y ve un aviso de error con el texto "El email o la contraseña no son correctos."

#### Scenario: Errores de formato junto al campo

- **WHEN** la persona envía un email con formato inválido y la contraseña vacía
- **THEN** ve "Introduce una dirección de email válida." bajo el campo Email y "Falta rellenar la contraseña." bajo el campo Contraseña

### Requirement: Pantalla de registro

La aplicación web SHALL ofrecer en `/register` un formulario "Crea tu cuenta" con los campos Nombre completo (marcado como opcional), Email, Contraseña (con la ayuda "Entre 8 y 32 caracteres.") y Repite la contraseña, un botón "Crear cuenta" y un enlace "Inicia sesión" que lleva al login. Al registrarse con éxito SHALL abrir la sesión y llevar a la persona a su perfil. Un nombre vacío o formado solo por espacios SHALL tratarse como ausencia de nombre.

#### Scenario: Registro correcto

- **WHEN** una persona sin sesión rellena el formulario con datos válidos y pulsa "Crear cuenta"
- **THEN** pasa a ver su pantalla de perfil ya con la sesión iniciada

#### Scenario: Nombre en blanco

- **WHEN** la persona se registra dejando Nombre completo vacío o con solo espacios
- **THEN** su perfil muestra "Sin nombre" como nombre

#### Scenario: Contraseñas distintas detectadas en pantalla

- **WHEN** la persona escribe contraseñas distintas en Contraseña y Repite la contraseña y pulsa "Crear cuenta"
- **THEN** ve "Las contraseñas no coinciden." bajo Repite la contraseña y no se envía nada al servidor

#### Scenario: Email ya registrado

- **WHEN** la persona intenta registrarse con un email que ya tiene cuenta
- **THEN** ve "Ese email ya está registrado. Inicia sesión en su lugar." bajo el campo Email

#### Scenario: Contraseña fuera de rango

- **WHEN** la persona envía una contraseña de menos de 8 caracteres
- **THEN** ve "la contraseña debe tener al menos 8 caracteres." bajo el campo Contraseña en lugar del texto de ayuda

#### Scenario: Envío en curso

- **WHEN** la persona pulsa "Crear cuenta" y la respuesta aún no ha llegado
- **THEN** el botón muestra "Creando cuenta…" y está deshabilitado

### Requirement: Errores de conexión y del servidor en los formularios

Los formularios de login y registro SHALL mostrar un aviso general cuando la petición falle por causas ajenas a los datos introducidos: "No se pudo conectar con el servidor. Comprueba que el backend está arrancado." si el servidor no responde, y "Algo ha ido mal en el servidor. Inténtalo de nuevo en un momento." ante cualquier otro error no previsto.

#### Scenario: Backend caído

- **WHEN** la persona envía el formulario de login con el backend apagado
- **THEN** ve el aviso "No se pudo conectar con el servidor. Comprueba que el backend está arrancado." y el botón vuelve a estar disponible

### Requirement: Pantalla de perfil

La aplicación web SHALL mostrar en `/profile` una tarjeta con las iniciales del usuario en un círculo, su nombre completo (o "Sin nombre" si no tiene), su email, la fecha "Miembro desde" en formato largo en castellano, y un botón "Cerrar sesión".

#### Scenario: Ver el perfil

- **WHEN** una persona con sesión y nombre "Ada Lovelace" abre `/profile`
- **THEN** ve "AL", "Ada Lovelace", su email y "Miembro desde" seguido de su fecha de alta en formato largo en castellano (p. ej. "28 de octubre de 2026"), según la zona horaria de su navegador

### Requirement: Cierre de sesión en la aplicación web

Al pulsar "Cerrar sesión", la aplicación web SHALL cerrar la sesión local de inmediato y llevar a la persona a `/login`, aunque el servidor no confirme la revocación del token.

#### Scenario: Cerrar sesión

- **WHEN** una persona con sesión pulsa "Cerrar sesión" en su perfil
- **THEN** pasa a ver la pantalla de login sin ningún aviso de error
- **AND** al recargar la página sigue sin sesión

#### Scenario: Cerrar sesión con el backend caído

- **WHEN** la persona pulsa "Cerrar sesión" con el backend apagado
- **THEN** igualmente pasa a ver la pantalla de login sin sesión
- **AND** el token que tenía sigue siendo válido en la API, porque nunca llegó a revocarse

### Requirement: Persistencia de la sesión entre recargas

La aplicación web SHALL conservar la sesión al recargar la página o volver a abrirla en el mismo navegador, comprobando contra la API que el token guardado sigue siendo válido antes de dar acceso. Mientras se comprueba SHALL mostrar un indicador de carga a pantalla completa en lugar de redirigir.

#### Scenario: Recarga con sesión válida

- **WHEN** una persona con sesión recarga `/profile`
- **THEN** ve brevemente un indicador de carga y después su perfil, sin pasar por el login

#### Scenario: Token rechazado al recargar

- **WHEN** la persona recarga la página y la API rechaza el token guardado
- **THEN** acaba en `/login` con el aviso "Tu sesión ha caducado. Vuelve a iniciar sesión."
- **AND** las siguientes recargas ya no intentan recuperar esa sesión

#### Scenario: Backend caído al recargar

- **WHEN** la persona recarga la página con el backend apagado
- **THEN** acaba en `/login` con el aviso "No se pudo conectar con el servidor. Comprueba que el backend está arrancado."
- **AND** al volver a recargar con el backend ya arrancado recupera su sesión sin volver a introducir credenciales

#### Scenario: Error del servidor al recargar

- **WHEN** la persona recarga la página y la API responde con un error del servidor (5xx)
- **THEN** acaba en `/login` con el aviso "Algo ha ido mal en el servidor. Inténtalo de nuevo en un momento."
- **AND** al volver a recargar con la API ya sana recupera su sesión

#### Scenario: El aviso de sesión perdida persiste hasta entrar

- **WHEN** la persona ve un aviso de sesión perdida en `/login` y envía un intento fallido cuyos errores se muestran solo junto a los campos
- **THEN** el aviso de sesión perdida sigue visible arriba, y solo desaparece cuando consigue iniciar sesión o cuando lo sustituye un aviso general del nuevo intento

### Requirement: Protección de rutas en la aplicación web

La aplicación web SHALL permitir el acceso a `/profile` solo con sesión, llevando a `/login` a quien no la tenga, y SHALL llevar a `/profile` a quien tenga sesión e intente abrir `/login` o `/register`. Cualquier otra dirección SHALL redirigir a `/profile`, y desde ahí se aplica la regla anterior.

#### Scenario: Perfil sin sesión

- **WHEN** una persona sin sesión abre `/profile`
- **THEN** pasa a ver la pantalla de login

#### Scenario: Login con sesión

- **WHEN** una persona con sesión abre `/login` o `/register`
- **THEN** pasa a ver su perfil

#### Scenario: Dirección desconocida

- **WHEN** una persona sin sesión abre `/cualquier-cosa`
- **THEN** acaba en la pantalla de login
- **AND** si tuviera sesión, acabaría en su perfil
