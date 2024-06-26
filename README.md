# Curso de AWS Redshift para Manejo de Big Data  
## 1. Que es un DataWarehouse:  
Es una base de datos muy grande con muchas fuentes de datos. Es una estructura Analítica que consolida toda la información de distintas fuentes.  

### ¿Como los cargo?
Para llevarse estos datos a un datawarehouse hacemos un ETL. Extraigo los datos de todas las fuentes, los transformo y luego los cargos dentro del DWH.  

### Arquitectura del DWH:
En general la industria cuenta con una arquitectura similar, usando el modelo dimensional. Teniendo dimensiones y una tabla de hechos con todas las buenas prácticas.  

### Modelo del curso:
El modelo que se trabajará a través de este repo, es un modelo de ventas relacionado de la siguiente forma:  
![](./images/AWS1.PNG)  

## 2. Bases de datos columnares y arquitectura orientada a optimización.  
Estas son bases de datos optimizadas para recuperación rápida de columnas enteras de datos. Existen otro tipo de arquitecturas basadas en filas, estas son más OLTP, están orientadas a lectura y escritura rápida basadas en filas. 

Lo que cambia en cada uno, es el bloque de datos. En filas, se va guardando a un bloque de datos en disco duro. Esto tiene sentido ya que busca el registro más rápido, por ejemplo al realizar una compra, los puntos que se van a actualizar correponde a un id de cliente. Estos soportan todas las operaciones y transacciones de la compañía.  

El problema es que normalmente cada bloque pesa 32 kilobytes. Para este tipo de tablas, si yo quisiera una consulta a una tabla de 10 columnas, y realmente requiero 3, internamente el OLTP lee las 7 restantes.  

En bases de datos columnares, cada columna se guarda en uno de los bloques. Redshift por su parte, ocupa 1 megabyte por cada bloque. Es decir, si quiero una columna solamente de n columnas, se accede mucho más rápido.  
  
En bases de datos columnares es muy dificil realizar actualizaciones si quedó algún error a nivel de registros.  
  
### Otras opciones:
- Google BigQuery,
- Snowflake
- Apache Hbase.  
  
Redshift por su lado, es altamente escabale y es fácil de integrar. Es la base de datos rápida y económica en la nube.  

## 3. Como funciona Redshift  
Redshift reparte el trabajo dentro de un cluster. Este cluster tiene varios nodos. Todas estas tareas ocurren en paralelo. Cada uno tiene su propia cpu, ram y tiene segmentos virtuales. Redshift está basada en postgres.

## 4. Hands-On Redshift [Carga y Creación]
Crearemos las tablas del curso con el script SQL brindado por el profesor. Una particularidad es que las tablas se definieron usando `sortkeys` y `distkeys`:
```SQL
create table users (
	userid integer not null distkey sortkey,
	username char(8),
	firstname varchar(30),
	lastname varchar(30),
	city varchar(30),
	state char(2),
	email varchar(100),
	phone char(14),
	likesports boolean,
	liketheatre boolean,
	likeconcerts boolean,
	likejazz boolean,
	likeclassical boolean,
	likeopera boolean,
	likerock boolean,
	likevegas boolean,
	likebroadway boolean,
	likemusicals boolean);  
```  
- `distkey` es una llave de distribución, distribuye los datos dentro del nodo por el `userid`. 

- `sortkey` es una llave de ordenamiento, lo cual va a permitir que siempre venga creado.

Ahora, luego de esto, lo siguiente es tomar del bucket de S3 los archivos proporcionados para el curso, dónde vamos a cargarlos mediante un copy. El Copy en redshift recibe los siguientes parámetros:
```SQL
copy [basededatos].[schema].[tabla] 
from 's3://[bucket]/[carpeta/archivo].extension'
credentials 'aws_iam_role=[ARN del Rol de IAM (Esto se encuentra en IAM)]'
delimiter '[tipo de separador]' timeformat '[opcional]' region '[region de trabajo]';
```  
Este copy toma todo el archivo de los parámetros específicados y los alimenta a la tabla en la base de datos de redshift.  

## 5. Comprensión en Redshift  
Las tablas deben comprimirse antes de la creación. La sentencia que se utiliza es la siguiente:  
```SQL
CREATE TABLE [nombre_tabla] (
    [nombre_columna] [tipo_dato] ENCODE encoding-type
);

-- Por ejemplo:

CREATE TABLE test_comprension (
    nombre varchar(30) ENCODE TEXT255
);
```  
### Algoritmos de compresión
- Existe la codificación Raw y esta es la base con la que cargamos las tablas.  

- Por su parte, la AZ64 es un algoritmo nativo de AWS para comprimir la información. Usa el SIMD (Single Instruction Multiple Data). Esto quiere decir que envía una única instrucción a múltiples partes de Redshift.  

- Codificación por diccionario de Bytes: Es eficaz cuando una columna tiene una cantidad limitada de valores únicos (menos de 256). Esto crea un diccionario en un bloque de redshift de 1 MB. Y cada bloque pasa de 210 bytes a 7 bytes solamente.

- Codificación Delta: es muy útil para datos numéricos. Viene de 1 y 2 bytes. En el rango de 1 byte, solo va desde -127 a 127 bits, si en tamaño supera esto, no se aplica. Por otro lado, en el rango de 2 bytes, va desde -32K a 32K. 

- Codificación LZO: Es útil para cadenas de texto libre.  

- Codificación Mostly: Se utiliza cuando la mayoría de los datos de una columna son mejores en bits al valor de la columna misma. Esta viene en compresiones de 8, 16 y 32 bits. Hay que conocer los datos para saber que codificación aplicar.

- Codificación Runlenght: Crea un token para datos que contiene solamente pocas repeticiones.  
  
- Text 255  Text32k: Son útiles para compromir columnas varchar en las que se repiten con frecuencia las mismas palabras.  
  
- Zstander: mueve bytes entre si para lograr la compresión. 

#### Ejemplo de una tabla habitual con las relaciones de compresión:  
![](./images/AWS2.PNG)  

## 6. Creando tablas para probar compresiones:
Un ejemplo de una tabla puede ser con el formato nombre, vamos a probar los distintos tipos de compresion de la siguiente forma:
1. Creamos la tabla que vamos a representar:  
```SQL
create table encoding_venue (
    nameraw         varchar(100) encode RAW,
    namebytedict    varchar(100) encode BYTEDICT,
    namelzo         varchar(100) encode LZO,
    namerunlenght   varchar(100) encode RUNLENGTH,
    nametext255     varchar(100) encode TEXT255,
    nametext32k     varchar(100) encode TEXT32K,
    namezstd        varchar(100) encode ZSTD
);
```
2. Insertamnos la tabla cartesiana que hicimos, dentro de la tabla encoding_venue:  
```SQL
insert into public.encoding_venue (
select	
	venuename, 
	venuename, 
	venuename, 
	venuename, 
	venuename,
	venuename,
	venuename
from cartesian_venue);
```

Luego de esto, cruzamos con las tablas del sistema (esto no aplica a serverless)