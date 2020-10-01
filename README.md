# Reto del curso de PostgreSQL Aplicado a Ciencia de Datos

# Platzi movies dashboard

## Top 10
Vamos a averiguar cuál es el top 10 de películas más rentadas.
```sql
select 
		peliculas.pelicula_id as id,
		peliculas.titulo,
		count(*) as numero_rentas,
		row_number () over (
			order by count(*) desc
		) as lugar
from	rentas
	inner join inventarios on rentas.inventario_id = inventarios.inventario_id
	inner join peliculas on inventarios.pelicula_id = peliculas.pelicula_id
group by peliculas.pelicula_id
order by numero_rentas desc
limit 10;
```
![Result](/images/result1.png)

## Actualizando precios

```sql
select	peliculas.pelicula_id,
		tipos_cambio.tipo_cambio_id,
		tipos_cambio.cambio_usd * peliculas.precio_renta AS precio_cop
from	peliculas,
		tipos_cambio
where	tipos_cambio.codigo = 'COP';
```
![Result](/images/result2.png)

Utilizando store proce
dures para obtener este resultado. Para ello, vamos a valernos del menu para crear el stored procedure.
1. Click derecho en "Trigger Functions / Create / Trigger Function"
![Primer paso](/images/1.png)
2. Le damos nombre a la trigger function y le asignamos el usuario que podrá ejecutarla.
![Segundo paso](/images/2.png)
3. En la pestaña "Definition" verificamos que el tipo de retorno sea una Trigger Function y el lenguaje que se utilizará.
![Tercer paso](/images/3.png)
4. Escribimos el código y le damos clic a Save.
![Cuarto paso](/images/4.png)

```sql
BEGIN
	insert into precio_peliculas_tipo_cambio(
		pelicula_id,
		tipo_cambio_id,
		precio_tipo_cambio,
		ultima_actualizacion
	)
	select	new.pelicula_id,
			tipos_cambio.tipo_cambio_id,
			tipos_cambio.cambio_usd * NEW.precio_renta as precio_tipo_cambio,
			current_timestamp
	from	tipos_cambio
	where	tipos_cambio.codigo = 'COP';
	return new;
END
```
**Creando el trigger**
```sql
create trigger trigger_update_tipos_cambio
	after insert or update
	on public.peliculas
	for each row
	execute procedure public.precio_peliculas_tipo_cambio();
```
Al modificar el primer precio renta de la primera película y guardarlo, se ejecuta el trigger y se actualiza la tabla.
![Result](/images/result3.png)


## Usando rank y percent rank

### Definiciones

#### La función **PERCENT_RANK()**
Es como la función CUME_DIST(). La PERCENT_RANK() evalúa la posición relativa de un valor dentro de un conjunto de valores.

A continuación, se ilustra la sintaxis de la PERCENT_RANK():
```sql
PERCENT_RANK() OVER (
    [PARTITION BY partition_expression, ... ]
    ORDER BY sort_expression [ASC | DESC], ...
)
```
En esta sintaxis:

**PARTITION BY:** divide las filas en varias particiones a las que la función PERCENT_RANK() es aplicada.
* La cláusula es opcional. Si lo omite, la función trata todo el conjunto de resultados como una sola partición.

**ORDER BY:** especifica el orden de las filas en cada partición a la que se aplica la función.

**Valor devuelto**: La función PERCENT_RANK() devuelve un resultado mayor que 0 y menor o igual que 1.

`0 < PERCENT_RANK() <= 1`

#### La **función RANK()** 
Asigna un rango a cada fila dentro de una partición de un conjunto de resultados.

Para cada partición, el rango de la primera fila es 1. La RANK() agrega el número de filas vinculadas al rango vinculado para calcular el rango de la siguiente fila, por lo que es posible que los rangos no sean secuenciales. Además, las filas con los mismos valores obtendrán el mismo rango.

A continuación, se ilustra la sintaxis de la RANK()función:
```sql
RANK() OVER (
    [PARTITION BY partition_expression, ... ]
    ORDER BY sort_expression [ASC | DESC], ...
)
```
En esta sintaxis:

la cláusula **PARTITION BY** distribuye filas del conjunto de resultados en particiones a las que la función RANK() es aplicada.

la cláusula **ORDER BY** especifica el orden de las filas en cada una de las particiones a las que se aplica la función.

*La Función RANK() puede ser útil para crear informes top-N y bottom-N*.

### PERCENT_RANK

```sql
select 
		peliculas.pelicula_id as id,
		peliculas.titulo,
		count(*) as numero_rentas,
		percent_rank () over (
			order by count(*) desc
		) as lugar
from	rentas
	inner join inventarios on rentas.inventario_id = inventarios.inventario_id
	inner join peliculas on inventarios.pelicula_id = peliculas.pelicula_id
group by peliculas.pelicula_id
order by numero_rentas desc;
```
![Percent_rank](/images/percent_rank.png)

### DENSE_RANK

```sql
select 
		peliculas.pelicula_id as id,
		peliculas.titulo,
		count(*) as numero_rentas,
		dense_rank () over (
			order by count(*) desc
		) as lugar
from	rentas
	inner join inventarios on rentas.inventario_id = inventarios.inventario_id
	inner join peliculas on inventarios.pelicula_id = peliculas.pelicula_id
group by peliculas.pelicula_id
order by numero_rentas desc;
```
![Dense_rank](/images/dense_rank.png)

## Ordenando datos geográficos

```sql
select	ciudades.ciudad_id,
		ciudades.ciudad,
		count(*) as rentas_por_ciudad
from	ciudades
	inner join direcciones on ciudades.ciudad_id = direcciones.ciudad_id
	inner join tiendas on tiendas.direccion_id = direcciones.direccion_id
	inner join inventarios on inventarios.tienda_id = tiendas.tienda_id
	inner join rentas on inventarios.inventario_id = rentas.inventario_id
group by ciudades.ciudad_id;
```
![Ciudades](/images/ciudades.png)

## Datos en el tiempo

Mostrando el número de rentas en una línea de tiempo, se ordenan primero el año, mes, y luego el título.
```sql
select	date_part('year', rentas.fecha_renta) as anio,
		date_part('month', rentas.fecha_renta) as mes,
		peliculas.titulo,
		count(*) as numero_rentas
from rentas
	inner join inventarios on rentas.inventario_id = inventarios.inventario_id
	inner join peliculas on peliculas.pelicula_id = inventarios.pelicula_id
group by anio, mes, peliculas.pelicula_id;
```
![Result4](/images/result4.png)

Ahora, si lo que queremos es ver cómo han ido evolucionando las rentas mes a mes, usamos el siguiente query.
```sql
select	date_part('year', rentas.fecha_renta) as anio,
		date_part('month', rentas.fecha_renta) as mes,
		count(*) as numero_rentas
from rentas
	inner join inventarios on rentas.inventario_id = inventarios.inventario_id
	inner join peliculas on peliculas.pelicula_id = inventarios.pelicula_id
group by anio, mes
order by anio, mes;
```
![Result5](/images/result5.png)

## Visualizando datos con Tableau
