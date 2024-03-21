# Gestión de una Biblioteca

`Postgres`

## Mandato

Una biblioteca necesita una base de datos para gestionar su inventario de libros, préstamos a los usuarios, y el seguimiento de los usuarios y sus multas. La biblioteca desea optimizar sus operaciones mediante el uso eficiente de `stored procedures`, `funciones`, `cursores`, `triggers`, expresiones de tabla común (`CTE`), operadores `HAVING`, entre otros.

## Requerimientos

### 1. Diseño de la Base de Datos

Crea un esquema de base de datos que incluya tablas para `libros`, `usuarios`, `préstamos` y `multas`. Define las relaciones entre estas tablas.
Indique las herramientas utilizadas.

#### Comandos

Creación de la `base de datos` para la biblioteca

```sql
CREATE DATABASE biblioteca;
```

Creación del `esquema` de la base de datos para la biblioteca

```sql
CREATE SCHEMA biblioteca;
```

`Tablas`

```sql
CREATE TABLE biblioteca.categorias (
  id_categoria SERIAL PRIMARY KEY,
  nombre_categoria VARCHAR(255) NOT NULL,
  descripcion TEXT
);

CREATE TABLE biblioteca.libros (
  id_libro SERIAL PRIMARY KEY,
  titulo VARCHAR(255) NOT NULL,
  autor VARCHAR(255) NOT NULL,
  anio_publicacion INT NOT NULL,
  inventario INTEGER NOT NULL DEFAULT 0,
  id_categoria INT NOT NULL REFERENCES biblioteca.categorias(id_categoria)
);

CREATE TABLE biblioteca.usuarios (
  id_usuario SERIAL PRIMARY KEY,
  nombre VARCHAR(100),
  correo VARCHAR(100)
);

CREATE TABLE biblioteca.prestamos (
  id_prestamo SERIAL PRIMARY KEY,
  id_libro INT NOT NULL REFERENCES biblioteca.libros(id_libro),
  id_usuario INT NOT NULL REFERENCES biblioteca.usuarios(id_usuario),
  fecha_prestamo DATE NOT NULL,
  fecha_devolucion DATE
);


CREATE TABLE biblioteca.multas (
  id_multa SERIAL PRIMARY KEY,
  id_prestamo INT NOT NULL REFERENCES biblioteca.prestamos(id_prestamo),
  monto DECIMAL(10, 2) NOT NULL,
  fecha_multa DATE NOT NULL,
  motivo VARCHAR(255) NOT NULL
);
```

### 2. Stored Procedures y Funciones

[Realiza el insert de datos primero](####`Inserts`)
- 2.1 Crea stored procedures para agregar, actualizar y eliminar libros y usuarios.

#### stored procedures `agregar_libro`

```sql
CREATE OR REPLACE PROCEDURE biblioteca.agregar_libro(
  IN titulo VARCHAR(255),
  IN autor VARCHAR(255),
  IN anio_publicacion INT,
  IN inventario INT,
  IN id_categoria INT
)
AS $$
BEGIN
  INSERT INTO biblioteca.libros (titulo, autor, anio_publicacion, inventario, id_categoria)
  VALUES (titulo, autor, anio_publicacion, inventario, id_categoria);
END;
$$ LANGUAGE plpgsql;
```

#### Ejemplo de uso

```sql
CALL biblioteca.agregar_libro('El Principito', 'Antoine de Saint-Exupéry', 1943, 69, 1);
```

#### stored procedures `actualizar_libro`

```sql
CREATE OR REPLACE PROCEDURE biblioteca.actualizar_libro(
  IN libro_id INT,
  IN nuevo_titulo VARCHAR(255),
  IN nuevo_autor VARCHAR(255),
  IN nuevo_anio_publicacion INT,
  IN nuevo_inventario INT,
  IN nuevo_id_categoria INT
)
AS $$
BEGIN
  UPDATE biblioteca.libros
  SET titulo = nuevo_titulo,
    autor = nuevo_autor,
    anio_publicacion = nuevo_anio_publicacion,
    inventario = nuevo_inventario,
    id_categoria = nuevo_id_categoria
  WHERE biblioteca.libros.id_libro = libro_id;
END;
$$ LANGUAGE plpgsql;
```

#### Ejemplo de uso

```sql
CALL biblioteca.actualizar_libro(1, 'Cien años de soledad', 'Gabriel García Márquez', 1967, 13, 3);
```

#### stored procedures `eliminar_libro`

```sql
CREATE OR REPLACE PROCEDURE biblioteca.eliminar_libro(
  IN libro_id INT
)
AS $$
BEGIN
  DELETE FROM biblioteca.libros
  WHERE biblioteca.libros.id_libro = libro_id;
END;
$$ LANGUAGE plpgsql;
```

#### Ejemplo de uso

```sql
CALL biblioteca.eliminar_libro(1);
```

#### stored procedures `agregar_usuario`

```sql
CREATE OR REPLACE PROCEDURE biblioteca.agregar_usuario(
  IN nombre VARCHAR(100),
  IN correo VARCHAR(100)
)
AS $$
BEGIN
  INSERT INTO biblioteca.usuarios (nombre, correo)
  VALUES (nombre, correo);
END;
$$ LANGUAGE plpgsql;
```

#### Ejemplo de uso

```sql
CALL biblioteca.agregar_usuario('Ana García', 'ana.garcia@correo.com');
```

#### stored procedures `actualizar_usuario`

```sql
CREATE OR REPLACE PROCEDURE biblioteca.actualizar_usuario(
  IN usuario_id INT,
  IN nuevo_nombre VARCHAR(100),
  IN nuevo_correo VARCHAR(100)
)
AS $$
BEGIN
  UPDATE biblioteca.usuarios
  SET nombre = nuevo_nombre,
    correo = nuevo_correo
  WHERE biblioteca.usuarios.id_usuario = usuario_id;
END;
$$ LANGUAGE plpgsql;
```

#### Ejemplo de uso

```sql
CALL biblioteca.actualizar_usuario(1, 'María López', 'maria.lopez@correo.com');
```

#### stored procedures `eliminar_usuario`

```sql
CREATE OR REPLACE PROCEDURE biblioteca.eliminar_usuario(
  IN usuario_id INT
)
AS $$
BEGIN
  DELETE FROM biblioteca.usuarios
  WHERE biblioteca.usuarios.id_usuario = usuario_id;
END;
$$ LANGUAGE plpgsql;
```

#### Ejemplo de uso

```sql
CALL biblioteca.eliminar_usuario(1);
```

- 2.2 Implementa una función para calcular el monto total de multas de un usuario.

Creación de la función

```sql
CREATE OR REPLACE FUNCTION biblioteca.calcular_monto_multas_usuario(usuario_id INT)
RETURNS DECIMAL(10, 2)
AS
$$
DECLARE
  total_multas DECIMAL(10, 2);
BEGIN
  SELECT SUM(monto) INTO total_multas
  FROM biblioteca.multas
  WHERE id_prestamo IN (
    SELECT id_prestamo
    FROM biblioteca.prestamos
    WHERE id_usuario = usuario_id
  );

  RETURN total_multas;
END;
$$
LANGUAGE PLPGSQL;
```

#### Ejemplo de uso

```sql
SELECT biblioteca.calcular_monto_multas_usuario(2);
```

#### `Inserts` de datos para poner a prueba el punto `2.2` (y para despues)

```sql
-- insert de datos para poder hacer la función para calcular el monto total de multas de un usuario
INSERT INTO biblioteca.categorias (nombre_categoria, descripcion)
VALUES
  ('Literatura clásica', 'Obras literarias consideradas como clásicos de la literatura universal.'),
  ('Ficción', 'Historias inventadas que no se basan en hechos reales.'),
  ('No ficción', 'Libros que se basan en hechos reales y proporcionan información sobre un tema específico.'),
  ('Ciencia ficción', 'Ficción que explora temas relacionados con la ciencia y la tecnología.'),
  ('Fantasía', 'Ficción que incluye elementos mágicos o sobrenaturales.'),
  ('Romance', 'Historias que se centran en las relaciones románticas entre dos personas.'),
  ('Misterio', 'Historias que giran en torno a un crimen o un enigma que debe ser resuelto.'),
  ('Aventura', 'Historias emocionantes que relatan viajes, exploraciones o hazañas heroicas.'),
  ('Autoayuda', 'Libros que ofrecen consejos y orientación para mejorar la vida personal.'),
  ('Humor', 'Libros que tienen como objetivo hacer reír al lector.'),
  ('Juvenil', 'Libros destinados a lectores jóvenes, generalmente con historias y personajes que les sean afines.');

INSERT INTO biblioteca.libros (id_libro, titulo, autor, anio_publicacion, inventario, id_categoria) VALUES
(1, 'El Principito', 'Antoine de Saint-Exupéry', 1943, 10, 2),
(2, 'Cien años de soledad', 'Gabriel García Márquez', 1967, 6, 4),
(3, '1984', 'George Orwell', 1949, 8, 3),
(4, 'El nombre de la rosa', 'Umberto Eco', 1980, 2, 1),
(5, 'El Gran Gatsby', 'F. Scott Fitzgerald', 2013, 69),
(6, 'Cien años de soledad', 'Gabriel García Márquez', 2011, 49, 5),
(7, 'Los juegos del hambre', 'Suzanne Collins', 2010, 100, 7),
(8, 'Ready Player One', 'Ernest Cline', 2012, 26, 3),
(9, 'El marciano', 'Andy Weir', 2014, 55, 6),
(10, 'Fourth Wing', 'Rebecca Yarros', 2023, 66, 8),
(11, 'Happy Place', 'Emily Henry', 2023, 99, 9),
(12, 'Pepito Perez', 'Emily Henry', 2023, 59, 3),
(13, 'Una mala vida en INTEC', 'F. Scott Fitzgerald', 2023, 666, 11);

INSERT INTO biblioteca.usuarios (id_usuario, nombre, correo) VALUES
(1, 'Ana García', 'ana.garcia@correo.com'),
(2, 'Juan Pérez', 'juan.perez@correo.com'),
(3, 'María López', 'maria.lopez@correo.com'),
(4, 'Sheztor Pollo', 'sheztor.pollo@correo.com');

INSERT INTO biblioteca.prestamos (id_prestamo, id_libro, id_usuario, fecha_prestamo, fecha_devolucion) VALUES
(1, 1, 1, '2024-02-15', '2024-03-11'),
(2, 1, 3, '2024-02-10', '2024-03-09'),
(3, 2, 1, '2024-01-01', '2024-03-18'),
(4, 2, 2, '2024-03-05', '2024-03-08'),
(5, 3, 2, '2024-02-22', NULL),
(6, 3, 3, '2024-03-15', NULL),
(7, 4, 3, '2024-03-12', '2024-03-17');

INSERT INTO biblioteca.multas (id_prestamo, monto, fecha_multa, motivo) VALUES
(1, 10.00, '2024-03-15', 'Devolución tardía'),
(1, 5.00, '2024-03-15', 'Libro dañado'),
(2, 15.00, '2024-03-15', 'Devolución tardía'),
(4, 15.00, '2024-03-15', 'Devolución tardía'),
(5, 7.50, '2024-03-15', 'Libro extraviado'),
(6, 7.50, '2024-03-15', 'Libro extraviado'),
(7, 2.50, '2024-03-15', 'Pérdida de marcador');

select * from biblioteca.libros
select * from biblioteca.usuarios
select * from biblioteca.prestamos
select * from biblioteca.multas
```

### 3. Cursores

- 3.1 Utiliza un cursor para calcular la cantidad total de libros prestados actualmente.

```sql
SELECT COUNT(*) AS "Total Libros Prestados"
FROM biblioteca.prestamos
WHERE fecha_devolucion IS NULL;
```

> [!NOTE]
> En postgres no hay cursores

### 4. Triggers

- 4.1 Crea un trigger que se active cuando se inserta un nuevo préstamo, y que actualice el inventario de libros disponibles.

`Función` que realiza la acción de actualizar el inventario

```sql
CREATE FUNCTION biblioteca.actualizar_inventario()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE biblioteca.libros
  SET inventario = inventario - 1
  WHERE id_libro = NEW.id_libro;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

`Trigger` que llamará la función que actualiza el inventario cuando se inserta un nuevo préstamo

```sql
CREATE TRIGGER actualizar_inventario
AFTER INSERT ON biblioteca.prestamos
FOR EACH ROW
EXECUTE PROCEDURE biblioteca.actualizar_inventario();
```

#### Ejemplo de acción

```sql
-- Insert de un prestamo
INSERT INTO biblioteca.prestamos (id_libro, id_usuario, fecha_prestamo)
VALUES (1, 2, '2024-03-18');

-- Consultar el inventario de ese libro prestado
SELECT id_libro, titulo, inventario
FROM biblioteca.libros
WHERE id_libro = 1;
```

### 5. Expresiones de Tabla Común (CTE)

- 5.1 Utiliza una `CTE` para obtener una lista de los usuarios con multas pendientes, junto con el monto total de las multas.

CTE llamada `MultasUsuario` que se une a las tablas `biblioteca.usuarios`, `biblioteca.prestamos` y `biblioteca.multas`

```sql
WITH MultasUsuario AS (
  SELECT
    u.id_usuario,
    u.nombre,
    SUM(m.monto) AS monto_total_multas
  FROM
    biblioteca.usuarios u
  JOIN -- Se realiza una unión con la tabla de préstamos utilizando el ID de usuario
    biblioteca.prestamos p ON u.id_usuario = p.id_usuario
  JOIN -- Se realiza otra unión con la tabla de multas utilizando el ID de préstamo
    biblioteca.multas m ON p.id_prestamo = m.id_prestamo
  WHERE
    p.fecha_devolucion IS NULL -- Filtrar préstamos no devueltos
  GROUP BY -- Se agrupa por el ID de usuario y nombre para calcular las sumas por usuario
    u.id_usuario, u.nombre
)
SELECT
  id_usuario,
  nombre,
  monto_total_multas
FROM
  MultasUsuario
WHERE
  monto_total_multas > 0; -- Filtrar usuarios con multas pendientes
```

### 6. Operadores

- 6.1 Utiliza el operador HAVING para encontrar los usuarios que tienen más de 3 libros prestados actualmente.

```sql
SELECT u.id_usuario, u.nombre, COUNT(*) AS libros_prestados
FROM biblioteca.usuarios u
INNER JOIN biblioteca.prestamos p ON u.id_usuario = p.id_usuario
GROUP BY u.id_usuario, u.nombre
HAVING COUNT(*) > 3;
```

- 6.2 Construya consultas y sentencias de reporteria. Luego responderla de modo que las mismas incluyan estos estos operadores

  - 6.2.1 `BETWEEN`

    ```sql
    -- Determinar cuántos libros se han publicado entre 2015 y 2020
    SELECT COUNT(*) AS total_libros
    FROM biblioteca.libros
    WHERE anio_publicacion BETWEEN 2015 AND 2020;
    ```

  - 6.2.2 `HAVING`

    ```sql
    -- Determinar que autores tienen 2 o más libros con un inventario mayor a 50?
    SELECT autor, COUNT(*) AS total_libros
    FROM biblioteca.libros
    GROUP BY autor
    HAVING COUNT(*) >= 2 AND AVG(inventario) > 50;
    ```

  - 6.2.3 `LIKE`

    ```sql
    -- Determinar cuales libros tienen un título que comienza con "El"
    SELECT titulo
    FROM biblioteca.libros
    WHERE titulo LIKE 'El%';
    ```

  - 6.2.4 `IN`

    ```sql
    -- Determinar cuales usuarios tienen préstamos con libros de Gabriel García Márquez
    SELECT nombre, correo
    FROM biblioteca.usuarios u
    INNER JOIN biblioteca.prestamos p ON u.id_usuario = p.id_usuario
    INNER JOIN biblioteca.libros l ON p.id_libro = l.id_libro
    WHERE l.autor IN ('Gabriel García Márquez');
    ```

  - 6.2.5 `EXISTS`

    ```sql
    -- Saber si hay usuarios que no tienen préstamos
    SELECT EXISTS (
      SELECT *
      FROM biblioteca.usuarios u
      LEFT JOIN biblioteca.prestamos p ON u.id_usuario = p.id_usuario
      WHERE p.id_usuario IS NULL
    );
    ```

  - 6.2.6 `DISTINCT`

    ```sql
    -- Saber cuántos autores distintos hay en la biblioteca
    SELECT COUNT(DISTINCT autor) AS total_autores
    FROM biblioteca.libros;
    ```

  - 6.2.7 `UNION`

    ```sql
    -- Saber que libros están prestados o tienen un inventario menor a 10
    SELECT DISTINCT titulo
    FROM biblioteca.libros l
    INNER JOIN biblioteca.prestamos p ON l.id_libro = p.id_libro
    UNION
    SELECT titulo
    FROM biblioteca.libros
    WHERE inventario < 10;
    ```

  - 6.2.8 Función de agregación: `MIN`, `Count`, `MAX`, entre otras.

    `MIN`

    ```sql
    -- Obtener el año de publicación más antiguo
    SELECT MIN(anio_publicacion) AS anio_minimo
    FROM biblioteca.libros;
    ```

    `MAX`

    ```sql
    -- Obtener el libro con mayor inventario
    SELECT l.titulo AS Titulo, l.inventario AS max_inventario
    FROM biblioteca.libros l
    WHERE l.inventario = (SELECT MAX(inventario) FROM biblioteca.libros);
    ```

    `COUNT`

    ```sql
    -- Contar el número total de préstamos
    SELECT COUNT(*) AS total_prestamos
    FROM biblioteca.prestamos;
    ```

    `AVG`

    ```sql
    -- Obtener el promedio de libros prestados por usuario
    SELECT AVG(prestamos_por_usuario) AS promedio_prestamos
    FROM (
      SELECT u.id_usuario, COUNT(*) AS prestamos_por_usuario
      FROM biblioteca.usuarios u
      INNER JOIN biblioteca.prestamos p ON u.id_usuario = p.id_usuario
      GROUP BY u.id_usuario
    ) AS t;
    ```

    `SUM`

    ```sql
    -- Obtener la suma total de los montos de multas en la tabla
    SELECT SUM(monto) AS suma_total_multas
    FROM biblioteca.multas;
    ```

## Tareas Adicionales (Claro, porque él quiere más)

- Implementa la lógica necesaria para que los libros tengan una fecha de devolución estimada, y que el sistema calcule automáticamente las multas si un libro se devuelve después de la fecha límite.

  ```sql
  -- Función que será utilizada para calcular la feha limite de devolución
  CREATE OR REPLACE FUNCTION biblioteca.calcular_fecha_devolucion(fecha_prestamo DATE)
  RETURNS DATE AS $$
  BEGIN
    RETURN fecha_prestamo + INTERVAL '13 days'; -- Tiene un maximo de 13 dias para entregar
  END;
  $$ LANGUAGE plpgsql;
  ```

  ```sql
  -- Función que será utilizada para calcular la feha limite de devolución
  CREATE OR REPLACE FUNCTION biblioteca.calcular_multa() RETURNS TRIGGER AS $$
  DECLARE
    fecha_limite DATE;
    cantidad_dias INTEGER;
  BEGIN
    fecha_limite := calcular_fecha_devolucion(NEW.fecha_prestamo);
    cantidad_dias := NEW.fecha_devolucion - fecha_limite;

    -- Verificar si ya existe una multa para el mismo préstamo con el mismo motivo
    IF EXISTS (
      SELECT 1
      FROM biblioteca.multas
      WHERE id_prestamo = NEW.id_prestamo
      AND motivo = 'Libro devuelto después de la fecha límite'
    ) THEN
      -- Si la fecha de devolución está dentro del rango de devolución,
      -- eliminar el registro de multa existente
      IF NEW.fecha_devolucion IS NULL OR NEW.fecha_devolucion <= fecha_limite THEN
        DELETE FROM biblioteca.multas
        WHERE id_prestamo = NEW.id_prestamo
        AND motivo = 'Libro devuelto después de la fecha límite';
      ELSE
        -- Si la fecha de devolución está fuera del rango de devolución,
        -- actualizar la fecha de la multa
        UPDATE biblioteca.multas
        SET fecha_multa = CURRENT_DATE,
        monto = cantidad_dias * 5
        WHERE id_prestamo = NEW.id_prestamo
        AND motivo = 'Libro devuelto después de la fecha límite';
      END IF;
    ELSE
        -- Si no existe, insertar una nueva multa
        IF NEW.fecha_devolucion IS NOT NULL AND NEW.fecha_devolucion > fecha_limite THEN
          INSERT INTO biblioteca.multas (id_prestamo, monto, fecha_multa, motivo)
          VALUES (NEW.id_prestamo,
            cantidad_dias * 5, -- 5 Pesos por día de diferencia
            CURRENT_DATE,
            'Libro devuelto después de la fecha límite');
        END IF;
    END IF;
    RETURN NEW;
  END;
  $$ LANGUAGE plpgsql;
  ```

  ```sql
  /*
    TRIGGER para que se ejecute la función para
    calcular la multa cuando s actualice la fecha de devolución en prestamo
  */

  CREATE TRIGGER trigger_calcular_multa
  AFTER UPDATE OF fecha_devolucion ON biblioteca.prestamos
  FOR EACH ROW EXECUTE FUNCTION biblioteca.calcular_multa();
  ```

- Considera la posibilidad de incluir una tabla de categorías para los libros y permite que los usuarios busquen libros por categoría.

  ```sql
  SELECT titulo, autor, anio_publicacion
  FROM biblioteca.libros
  WHERE id_categoria = 2;
  ```

- Diseña consultas que muestren los libros más populares, los usuarios con más multas, etc.

  ```sql
  -- Libros más populares
  SELECT titulo, autor, COUNT(*) AS prestamos
  FROM biblioteca.prestamos
  INNER JOIN biblioteca.libros ON biblioteca.prestamos.id_libro = biblioteca.libros.id_libro
  GROUP BY biblioteca.libros.id_libro
  ORDER BY prestamos DESC
  LIMIT 10;
  ```

  ```sql
  -- Usuarios con más multas
  SELECT nombre, correo, SUM(monto) AS total_multas
  FROM biblioteca.multas
  INNER JOIN biblioteca.prestamos ON biblioteca.multas.id_prestamo = biblioteca.prestamos.id_prestamo
  INNER JOIN biblioteca.usuarios ON biblioteca.prestamos.id_usuario = biblioteca.usuarios.id_usuario
  GROUP BY biblioteca.usuarios.id_usuario
  ORDER BY total_multas DESC
  LIMIT 10;
  ```

## Entrega

Los estudiantes deberán entregar un `script de SQL` que contenga la creación de las tablas, los `stored procedures`, `funciones`, `triggers`, `cursores`, `CTEs` y consultas necesarias para satisfacer los requerimientos mencionados anteriormente. Además, deben proporcionar ejemplos de uso de cada uno de estos elementos.
