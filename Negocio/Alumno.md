Usuario creado directamente por una [[Sede]] del tipo escolar — nunca se autorregistra.

# Alta
Lo crea el BranchAdmin o un librarian de esa sede escolar, individual o por lote (.csv / Excel, *pendiente* de implementar). Al crearlo:
- Correo electrónico (institucional).
- Contraseña generada aleatoriamente, que se envía por correo (*pendiente*: conectar el envío) y que el Student puede cambiar después.
- Avatar preestablecido (no elige el propio).

No necesita [[Identidad|verificación de identidad]] — a diferencia del [[Lector|User]] común, no hace falta DNI ni fecha de nacimiento para operar.

# Qué ve
- No ve el [[Foro]].
- Ve una sección de libros recomendados, curada por los librarians/BranchAdmin de su propia sede (no de toda la [[Red]]).

# Implementación
Rol `student` en [[Autenticación]]. Siempre tiene `sedeId` seteado a la sede escolar que lo creó — es ese campo el que determina qué sede le arma las recomendaciones y quién puede gestionarlo (su propia sede, no NetAdmin ni otra sede).
