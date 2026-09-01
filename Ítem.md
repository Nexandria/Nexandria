Cada ítem es un elemento existente en la biblioteca, pueden ser Libros, u otros.
# [[Libros]]
# [[Otros]]

Pero todos tienen algunas cosas en común:
# [[Nombre]]
Todos los elementos tienen un Nombre obligatoriamente, puede ser el título del libro, o puede ser el nombre general del ítem ej: Computadora SarmientoBA.
# [[Código de Barras]]
Cada ítem tiene un código de barras, que lo identifica como elemento, pero siempre se prioriza el que venga ya en el elemento, por ej. el ISBN.
No existe problema en el que existan códigos de barra iguales, siempre y cuando sean elementos iguales.

Formato: EAN-13
# [[Repetidos]]
Los elementos repetidos son aquellos que son o iguales de código de barras, o de estructura, es decir, es el mismo libro, pero diferente editorial, por lo tanto diferente códigos de barras.

Por ejemplo, para los objetos del tipo Otros, que suelen ser Computadoras, se usan las mismas estructuras, pero son items diferentes, en ese caso se da la opcion de generar un nuevo Código de Barras, que es único en el sistema.
Para ejemplares repetidos, simplemente se suma un nuevo ejemplar con la misma info.