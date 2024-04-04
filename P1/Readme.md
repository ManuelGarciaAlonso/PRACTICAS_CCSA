# Documentación de la Práctica: Despliegue de Servicio ownCloud

## Entorno de Desarrollo y de Producción
Para la práctica, ell entorno de desarrollo ha sido el siguiente:

- **Sistema Operativo**: Microsoft Windows 11 Pro.
- **Hardware**:
  - **Procesador**: Intel® Core™ i7-9750H CPU @ 2.60GHz.
  - **Memoria RAM**: 16 GB (15,8 GB utilizable).
- **Docker Desktop**: Versión 4.25.2.
- **Docker Compose**: Versión v2.23.0-desktop.1.

## Descripción de la Práctica y Problema a Resolver
El objetivo de esta práctica es la creación de un entorno de servicios que nos permita almacenar y acceder a archivos y hacer esto de forma segura, consistente y eficiente. Para ello vamos a usar ownCloud, que es una plataforma genial de código abierto para almacenar y
compartir archivos. Además, para la práctica debemos gestionar usuarios y autenticar sus contraseñas (usando LDAP), mantener la base de datos al día (usaremos MariaDB), hacer que todo vaya más rápido con una caché (Redis). Finalmente,
para asegurarnos de que si un servidor se cae, otro pueda seguir funcionando, haremo uso de HAProxy.


## Servicios Desplegados y su Configuración
Los servicios implementados y configurados en la práctica son:

1. **Servicio web ownCloud**: Es nuestro servidor de archivos.  Está configurado para correr en dos puertos para que podamos tener dos instancias en marcha al mismo tiempo, para tener alta disponibilidad (si uno se cae, el otro sigue y los usuarios no lo notan).

2. **MariaDB**: MariaDB es donde se guarda toda la información sobre los archivos y los usuarios. Se ha configurado con una arquitectura maestro-esclavo para asegurar persistencia y escalabilidad de los datos.

3. **Redis**: Utilizado como almacenamiento en caché para mejorar el rendimiento del servicio ownCloud y que no deba acceder a la base de datos constantemente.

4. **OpenLDAP**: Este es un servicio crucial para la práctica ya que sirve para la autenticación de usuarios, con la configuración necesaria para la gestión de los mismos y la seguridad del servicio.

5. **HAProxy**: Por ltimo, HAProxy se encarga de repartir la carga para que no se sature ningún servidor de onwCloud.

(Para cada servicio, se debe incluir una explicación detallada de las configuraciones y el archivo `docker-compose.yml` utilizado en la práctica, así como cualquier otro archivo de configuración relevante.)

###Configuración Específica

#### HAProxy
El servicio `haproxy` se configura para actuar como un balanceador de carga y un proxy inverso. Escucha en los puertos 80 y 8404 y redirecciona el tráfico hacia el servicio `ocserver`, que es el servidor de ownCloud.

- **Imagen**: Se utiliza `haproxytech/haproxy-alpine` para un balanceo de carga eficiente y ligero.
- **Red**: Conectado a la red `mynet`.
- **Volúmenes**: Se monta el directorio local `./haconfig` al directorio de configuración de HAProxy dentro del contenedor para cargar la configuración personalizada.
- **Dependencias**: Depende del servicio `ocserver` para el funcionamiento.

#### ownCloud Server
El `ocserver` es una instancia del servicio ownCloud que proporciona la interfaz web y el acceso a los archivos.

- **Imagen**: Se usa la versión especificada por la variable `${OWNCLOUD_VERSION}` de ownCloud.
- **Red**: Conectado a la red `mynet`.
- **Puertos**: Los puertos 8080 y 8081 están expuestos y mapeados al puerto 8080 del contenedor.
- **Volúmenes**: Se monta un volumen local para la persistencia de datos.
- **Variables de Entorno**: Se definen todas las variables necesarias para la configuración, incluyendo el dominio, base de datos y credenciales de administrador.
- **Replicación**: Se configura para tener 2 réplicas para alta disponibilidad.

#### MariaDB (Maestro y Esclavo)
Se configuran dos servicios `mariadb_master` y `mariadb_slave` para la replicación de la base de datos, lo que asegura la continuidad del servicio y la persistencia de los datos.

- **Imagen**: Ambos usan la imagen `mariadb`.
- **Red**: Conectados a la red `mynet`.
- **Volúmenes**: Se montan volúmenes locales para persistencia y configuraciones específicas para cada instancia.
- **Variables de Entorno**: Se establecen las credenciales y nombres de la base de datos.
- **Comandos**: Se especifican opciones de arranque para la replicación y el tamaño de paquetes.

#### Redis
`redis` se utiliza como almacenamiento en caché para mejorar el rendimiento del servicio ownCloud.

- **Imagen**: Se usa la imagen `redis`.
- **Red**: Conectado a la red `mynet`.
- **Volúmenes**: Se monta un volumen local para la persistencia de datos.
- **Comando**: Se especifica el número de bases de datos.

#### OpenLDAP
`openldap` proporciona servicios de autenticación de usuarios.

- **Imagen**: Se utiliza `osixia/openldap:latest`.
- **Red**: Conectado a la red `mynet`.
- **Puertos**: Se exponen los puertos 389 y 636 para el servicio LDAP y LDAP sobre TLS respectivamente.
- **Volúmenes**: Se montan volúmenes para los certificados y la configuración de LDAP.
- **Variables de Entorno**: Se configuran las credenciales de administrador, la organización y el dominio de LDAP.

#### phpLDAPadmin
`phpldapadmin` es una interfaz web para la administración de OpenLDAP.

- **Imagen**: Utiliza `osixia/phpldapadmin:latest`.
- **Red**: Conectado a la red `mynet`.
- **Puertos**: El puerto 7777 del host se mapea al puerto 80 del contenedor para acceder a la interfaz web.
- **Variables de Entorno**: Configura el host LDAP y deshabilita HTTPS para la interfaz.
- **Dependencias**: Depende del servicio `openldap`.

### Redes
Se ha definido una red llamada `mynet` de cara a la comunicación entre servicios.

Fianlmente, todos los servicios está configurados para reiniciarse automáticamente con `restart: always`.



## Instrucciones para la Provisión de Servicios
Para desplegar los servicios de la práctica, se debe ejecutar únicameente el siguiente comando:
```bash
docker-compose up -d
```
Esto es gracias al archivo de configuración [docker-compose.yml](./docker-compose.yml) que ya lo tiene todo listo para arrancar. Debemos asegurarnos de estar en el directorio correcto y de tener Docker Desktop corriendo
[]()
[]()

Una vez hemos ejecutado el comando satistfactoriament podemos ver que los servicios están corriendo. Sin embargo, si comprobamos el estado de la replicación del servidor vemos que la base de datos esclava no está copiando a la maestra.
[Captura2]()

Para arreglar este problema debemos hacer una serie de modificaciones:
1. Archivo de configuración del maestro ([50-server_master.cnf](./50-server_master.cnf))
Añadimos las siguientes sentencias:
```md
  [mysqld]
	skip-name-resolve
	log-bin = 1
	binlog_do_db=owncloud
```
2. Archivo de configuración del esclavo ([50-server_slave.cnf](./50-server_slave.cnf))
Añadimos las siguientes sentencias:
```md
[mysqld]
	skip-name-resolve
	log-bin = 1
	binlog_do_db=owncloud
```
3. Archivo de configuración del despliegue de servicios ([docker-compose.yml](./docker-compose.yml))
- Añadir "- MYSQL_DATABASE=owncloud" al servicio *mariadb_slave* para que se muestre la base de datos owncloud en el esclavo.
- Añadir a los comandos de las bases de datos maestro y esclavo de mariadb las opciones correspondientes:
  * Maestro: "--log-bin", "--server-id=1", "--log-basename=master1", "--binlog-format=mixed"
  * Esclavo: "--server-id=2"
 

Para ir probando los cambios, debemos reiniciar el entorno para volver a probar, y para ello debemos ejecutar:
```bash
docker-compose down -v
docker system prune
```

## Conclusiones
Después de sumergirme en la práctica de montar un servicio de ownCloud con la magia de Docker y Docker Compose, he aprendido un par de cosas fundamentales que podrían ser útiles para cualquiera que se embarque en esta aventura. Aquí van algunas reflexiones finales:

- La importancia de la persistencia de datos: Al principio no era consciente de la necesidad de la persistencia de datos en los servicios de bases de datos como MariaDB. Hacer que la información sobreviva más allá del ciclo de vida de un contenedor es clave. Añadir la variable - MYSQL_DATABASE=owncloud en mariadb_slave fue un cambio pequeño pero crucial para asegurar la visibilidad de la base de datos.

- Si algo falla, vuelve al principio: Los comandos docker down y docker prune se convirtieron en mis mejores amigos. Hacer una limpieza y empezar de nuevo me ayudó a resolver problemas que parecían complicados, pero que eran el resultado de configuraciones pasadas conflictivas. Estos comandos fueron como un botón de reinicio que me permitió probar nuevas configuraciones desde un estado limpio.

- El diablo está en los detalles de la configuración: Un cambio aparentemente insignificante en los archivos de configuración puede tener un gran impacto. Ajustar las configuraciones de master y slave para la replicación en 50-server_master.cnf y 50-server_slave.cnf, y luego reflejar esos cambios en docker-compose.yml, fue fundamental para lograr la replicación efectiva y la alta disponibilidad.

- Identificación única y seguimiento: Asegurarse de que cada instancia de MariaDB tuviera su propio server-id y configurar adecuadamente el binlog fue una lección en ajustes finos. Eso demostró ser vital para la sincronización y el seguimiento efectivo de las operaciones entre la base de datos principal y la de respaldo.

- Los cambios pequeños pueden solucionar grandes problemas: A veces, una solución no requiere de rehacer todo desde cero, sino de pequeñas modificaciones. Ajustar los parámetros de log-bin y binlog_do_db fue todo lo que se necesitó para optimizar el proceso de replicación y rendimiento de la base de datos.

- Documentación y comunidad son salvavidas: No puedo enfatizar lo suficiente cuánto me apoyé en la documentación oficial y en la comunidad. Cada vez que me quedé atascado, había un foro o una guía que señalaba en la dirección correcta.

- En resumen, este proyecto fue un desafío pero increíblemente gratificante. Ha sido una prueba de que con las herramientas correctas y una comunidad de apoyo, incluso las tareas más intimidantes se pueden manejar de forma eficiente y efectiva. La nube de ownCloud que monté no es solo un espacio para archivos, sino un testimonio del aprendizaje continuo y de la capacidad de adaptación en el campo siempre cambiante de la tecnología.

