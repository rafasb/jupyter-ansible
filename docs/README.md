# Contenedor para ejecutar Ansible con Jupyter

Permite emplear la potente herramienta de Ansible, con una colección de plugins orientada a la automatización de equipos de red y a través de la interfaz web de Jupyter

## Creación de la imagen

Esta solución nos permite crear una imagen personalizada para poder emplear ansible, los plugins e incluso inventarios y variables personalizados.

Los datos de partida del contenedor se ubican en el directorio `build-config` en el que tenemos la siguiente estructura:

directorio | fichero  | uso
---|---|---

-- | ansible.cfg | Parametrización de Ansible y plugins
-- | requierements.txt | Elementos de python a incluir en el contenedor
-- | ssh_config.conf | Personalización del acceso por SSH (Necesario en conmutadores viejos)
inventory |  | Contiene un inventario de equipos y variables
inventory | hosts.ini | Contiene el inventario de los hosts y como se agrupan
inventory/group_vars |  | Contiene ficheros con las variables de cada grupo. El nombre debe ser exactamente el nombre del grupo
inventory/group_vars | all | Variables definidas para todos los hosts. Se sobreescriben si eisten variables para el grupo o host
inventory/host_vars |  | Contiene ficheros con las variables de cada host. El nombre debe ser exactamente el nombre del host

* No se pueden cambiar los nombres de los ficheros que contienen extensión, ya que son básicos para la configuración del contenedor.
* No se pueden cambiar los nombres de los directorios.
* Se pueden añadir nuevos ficheros y cambiar el contenido de los ficheros existentes para personalizar el contenedor.
* No es necesario cargar el inventario y las variables, ya que en ejecución es posible incluir otros inventarios y variables, así como playbooks.
* En el servicio de jupyter existirán como invariables los valores introducidos a través del directorio `inventory`, pero será posible trabajar con otros inventarios o playbooks cargados a través del directorio común (`local`)

**Creación de la imágen**

En el directorio ejecutamos la orden siguiente:
```bash
docker build -t jupyter:latest .
```

Esto creará una imagen de docker con el nombre **jupyter** y el tag **latest**. Podemos cambiar la etiqueta `:latest` por otra para identificar la versión. Esta imagen se puede emplear para crear un contenedor que puede emplearse desde el CLI para ejecutar Ansible.

### Uso del contenedor en CLI

Para poder usarla de forma sencilla proponemos incluirla como alias con la siguiente instrucción:

```bash
echo alias da=\'docker run -ti --rm --name ansible_cli jupyter:latest\' >> ~/.bash_aliases
```

Esta instrucción nos permiten emplear el comando da de dos maneras. Simplemente da sin argumentos, leerá el valor de la clausula CMD del Dockerfile y ejecutará este comando. Es decir, entraremos en el contenedor con el shell bash. Así podremos ejecutar instrucciones de ansible o de sistema desde el contenedor.

También podemos usar el contenedor como una aplicación. A la instrucción de ansible le añadimos da y ejecutará el comando como si estuviera instalado en nuestro sistema. Por ejemplo:

```bash
da ansible all -m ping
```

En este caso verificará la conectividad con todos los host del inventario cargado en la imagen. Antes de poder usar el contenedor, debemos conocer como almacenar las contraseñas o algunas variables que no deben quedar expuestas.

### Cifrado de contraseñas para el inventario. (Uso del contenedor en CLI)

Para evitar que todas las contraseñas queden expuestas en los ficheros de configuración en texto plano usaremos el sistema de **vault** de Ansible. 

1) Cifrado de contraseñas. Usaremos el siguiente comando. Solicitará dos veces la **contraseña de cifrado de valores**:
`da ansible-vault encrypt_string 'valor_secreto' --name 'nombre_variable'`

2) Repetimos esta instrucción con los valores que deseamos cifrar, usando "siempre" la mismas **contraseña de cifrado de valores** (clave del vault)

2) Copiamos el resultado en el valor de la variable. Por ejemplo, en las variables del host o las variables del grupo **all** introduciremos la siguiente variable. En este ejemplo `nombre_variable` es `ansible_password`:
```yaml
ansible_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          35323065373032336462633938366233613935613132393462326364613836383666376632333137
          3232616262346534653432313038653363613339356137360a313662613833396538626462343632
          33363438623636636261303263373034643362666362326532303932376265353666336538636532
          6438616232346333300a646466386437666131653836323437313133343735643163623933353139
          6666
```

Al ejecutar las instrucciones de ansible, al comando añadimos la opción --ask-vault-password que solicitará la **contraseña de cifrado de valores** empleada para el cifrado de las contraseñas en el paso 1.

El comando de verificación de conectividad quedaría así:
```bash
da ansible all -m ping --ask-vault-password
```

## Inicio y uso de la imagen de jupyter notebook con ansible

Para poder arrancar el contenedor bastará con la instrucción siguiente:

```bash
docker-compose up -d
```

Usaremos el token en establecido en el fichero docker-compose.yml con la variable `JUPYTER_TOKEN`. Accedermos a la url http://\<IPdelHost\>:8888

Por supuesto, es parametrizable el puerto, el token y los directorios.

## Uso del contenedor con otros inventarios, playbooks y variables

En el entorno de jupyter disponemos del directorio `local` de manera que este permanecerá en el equipo entre reinicios. Este directorio nos permite crear inventarios alternativos, playbooks y alterar las varibles.

Si hay alguna variable cifrada en el inventario, que requiera del flag `--ask-vault-pass`, no funcionará en jupyter, ya que jupyter no permite interactividad del shell en los notebooks.

## Opciones de interés

En el directorio local se incluye un **notebook** de jupyter de ejemplo de uso.

## Referencias

https://docs.python.org/3/tutorial/venv.html

https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html

https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html#tip-for-variables-and-vaults

