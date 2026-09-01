El usuario objetivo del sistema.
# Fundamentos
- Existen 2 tipos: normal / alumno
- Tienen una cuenta común (avatar, nombre, apellido, edad)
- Tienen un puntaje (por y para los préstamos)
# Registro
**ESTA PARTE ES FUNDAMENTAL PORQUE CAMBIA LA EXPERIENCIA DE USUARIO**}

Existen diferentes tipos de usuario, por lo tanto puede que algunos se registren a posta, mientras que otros son registrados. Por ej: Alumno nace desde una sede, mientras que el usuario nace desde la red, con registro común.

Ahora, los usuarios comunes se registran usando BetterAuth con Google o Facebook y si bien tienen acceso al sistema, no es total.

- Correo electrónico
- Contraseña
- Nombre de usuario
- Avatar (opcional)

Puede ver sedes. buscar libros, pero no puede interactuar con los préstamos, ni con el foro.
## Verificación de Identidad
La app requiere que los usuarios comunes que quieren acceder a las funciones protegidas escaneen  el PDF417 que está en el DNI Argentino, extrayendo datos en este formato:
TRAMITE@APELLIDO@NOMBRE@SEXO@NRODNI@EJEMPLAR@NACIMIENTO@VENCIMIENTO

Con esos datos agregamos en los datos personales:
- Nro Documento (tiene que ser único en el sistema)
- Nombre y Apellido 
- Fecha Nacimiento (solo aceptamos mayores de edad)

**TODOS LOS USUARIOS QUE NO SEAN DEL TIPO COMÚN, SON CREADOS BAJO UN CORREO EN EL PANEL Y SE LES CREA UNA CONTRASEÑA ALEATORIA QUE SE ENVIA AL CORREO Y QUE ES CAMBIABLE (ej: AdminSed, bibliotecario, alumno).**
# [[Alumno]]
Usuario directamente creado por una sede del tipo escolar. Estas cuentas se generan con una contraseña aleatoria y se envian por correo cuando el AdminSed de la escolar lo registra.
Se pueden registrar usuarios de manera masiva por un .csv o excel.

No hace falta que estos usuarios tengan registro de identidad, simplemente:
- Correo Electrónico (institucional)
- Contraseña (generada aleatoriamente pero cambiable)
- Tienen un avatar preestablecido
- No ven el foro, lo que ven es una sección de libros recomendados por la institución que tambien lo manejan los bibliotecarios y AdminSed de esa sede.
# [[Algoritmo]]
Cada usuario arma un algoritmo personal al estilo TikTok en el que se recomiendan libros y se genera un algoritmo para tratar de llegar al libro objetivo. Entre los libros que likea, los que lee, los que muestra interés. 
# [[Identidad]]
Cada usuario puede generar una identidad temporal para que sea reconocido en el sistema. Por ej. el QR que genera Mercado Pago para pagar el colectivo. Genera un item en Redis que relaciona un codigo de 8 digitos alfanúmericos y un qr que si se presenta fisicamente se puede:
- Realizar un prestamo desde 0 
- Activar una reserva
- Ver historial
Todas estas acciones las realiza un bibliotecario desde su sistema.
# [[Reservas]]
El usuario puede reservar libros (que esten disponibles) para luego presentarse físicamente y dar como iniciado del préstamo, la reserva dura 3días hábiles y luego el libro vuelve a estar disponible. En la reserva se declara el préstamo que se quiere realizar: títulos y duración.
# [[Buscar]] Libros
Los usuarios pueden buscar títulos horizontalmente, es decir en todas las sedes de la red, pero también pueden buscar o ver los titulos de una sede particular. Los usuarios pueden ver las sedes en el mapa para elegir donde pueden realizar la reserva, o buscar directamente el titulo.