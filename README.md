# ProyectoFinal_Telematicos

# Integrantes

* Johan Sebastian Quimbayo Azcarate

* Jonathan Steven Narvaez Navia

* Simon Colonia Amador

* Juan David Moreno Mañunga

# Entregable a subir: Informe en formato IEEE, máx. 7 páginas.
§ Se sugiere una estructura del informe como la siguiente:

• Introducción

• Contexto (Problema)

• Alternativas de solución

• Diseño de la solución

• Implementación

• Pruebas

• Discusión de las pruebas

• Conclusiones

# PASO A PASO

## Consideraciones Previas Generales:
*   **Virtualización:** Necesitarás un entorno de virtualización como VirtualBox con Vagrant, o similar.
*   **Sistema Operativo Base:** Todas las máquinas virtuales usarán Ubuntu 22.04.1.
*   **Red:** Configura una red privada para que todas las VMs puedan comunicarse entre sí. Es crucial asignar IPs fijas o tener un sistema de resolución de nombres (como editar `/etc/hosts` en cada VM) para facilitar la comunicación.

---

## Paso a Paso Conceptual:

### Fase 0: Preparación Inicial Común a Todas las VMs
1.  **Creación de las VMs:** Crea cinco máquinas virtuales con Ubuntu 22.04.1. Asigna nombres descriptivos (ej: `mysql-master`, `mysql-slave1`, `mysql-slave2`, `mysql-router`, `mysql-client`).
2.  **Configuración de Red Básica:** Asegura que cada VM tenga una IP en la red privada y pueda conectarse con las demás.
3.  **Actualización del Sistema:** En cada VM, actualiza los paquetes del sistema operativo.

### Fase 1: Configuración de la VM Master (Servidor MySQL Primario Inicial)
1.  **Instalación de Software:** Instala el servidor MySQL y la herramienta MySQL Shell.
2.  **Configuración de MySQL para Group Replication:**
    *   Edita el archivo de configuración principal de MySQL (`mysqld.cnf`).
    *   Establece un `server_id` único.
    *   Define los parámetros específicos de Group Replication:
        *   `group_replication_group_name`: Un identificador único para tu clúster (será el mismo en todos los nodos).
        *   `group_replication_local_address`: La IP y puerto (ej: `IP_MASTER:33061`) que este nodo usará para la comunicación del clúster.
        *   `group_replication_group_seeds`: Inicialmente, solo la dirección del propio master (ej: `IP_MASTER:33061`). Más adelante se actualizará.
        *   `group_replication_start_on_boot`: Usualmente `off`, ya que AdminAPI lo maneja.
        *   `group_replication_bootstrap_group`: Usualmente `off`, ya que AdminAPI lo maneja.
3.  **Reinicio del Servicio MySQL:** Aplica los cambios reiniciando el servicio MySQL.
4.  **Inicialización del Clúster con MySQL Shell (AdminAPI):**
    *   Conéctate a la instancia MySQL usando MySQL Shell.
    *   Ejecuta `dba.configureInstance()` para preparar la instancia para Group Replication. Esto puede implicar crear un usuario administrador para el clúster (ej. `clusteradmin` como se menciona en el PDF).
    *   Ejecuta `dba.createCluster()` para crear el clúster InnoDB. Dale un nombre al clúster (ej. `myInnoDBCluster`) y especifica una lista de IPs permitidas (`ipAllowlist` o `ipWhitelist`).
5.  **Creación de Usuario para Aplicaciones:** Crea un usuario MySQL (ej. `cliente`) que las aplicaciones usarán para conectarse a través del Router.

### Fase 2: Configuración de las VMs Esclavas (Servidores MySQL Secundarios/Réplicas) - Repetir para Esclava 1 y Esclava 2
1.  **Instalación de Software:** Instala el servidor MySQL y MySQL Shell en cada VM esclava.
2.  **Configuración de MySQL para Group Replication:**
    *   Edita el archivo `mysqld.cnf` en cada esclavo.
    *   Establece un `server_id` único para cada esclavo (diferente del master y del otro esclavo).
    *   Define los parámetros de Group Replication:
        *   `group_replication_group_name`: El mismo que el del master.
        *   `group_replication_local_address`: La IP y puerto de este esclavo (ej: `IP_ESCLAVA1:33061`).
        *   `group_replication_group_seeds`: Una lista de las direcciones de todos los miembros potenciales del clúster (ej: `IP_MASTER:33061`,`IP_ESCLAVA1:33061`,`IP_ESCLAVA2:33061`).
        *   `group_replication_start_on_boot` y `group_replication_bootstrap_group` en `off`.
3.  **Reinicio del Servicio MySQL:** Aplica los cambios en cada esclavo.
4.  **Añadir Esclavos al Clúster (desde el Master usando MySQL Shell):**
    *   Conéctate a MySQL Shell en la VM Master, usando el usuario administrador del clúster.
    *   Obtén el objeto del clúster (`dba.getCluster()`).
    *   Para cada esclavo, ejecuta `cluster.addInstance()`, proporcionando la dirección del esclavo y las credenciales del usuario administrador del clúster. Especifica un método de recuperación (como `'clone'`) para que el esclavo se sincronice.
    *   Verifica el estado del clúster (`cluster.status()`) para confirmar que los esclavos se unen como `SECONDARY` y están en modo `super_read_only=ON`.

### Fase 3: Configuración de la VM Router (Nodo MySQL Router)
1.  **Instalación de Software:** Instala MySQL Router.
2.  **Bootstrap de MySQL Router:**
    *   Ejecuta el comando `mysqlrouter --bootstrap`. Deberás proporcionar la dirección de un miembro del clúster (ej. el master) y las credenciales del usuario administrador del clúster.
    *   Indica un directorio donde se guardará la configuración del router y un nombre de usuario para el propio router.
    *   Este proceso conecta el router al clúster, obtiene la topología (quién es primario, quiénes son secundarios) y genera un archivo de configuración (`mysqlrouter.conf`). Este archivo define los puertos de escucha para tráfico de lectura/escritura (ej. `6446`) y solo lectura (ej. `6447`).
3.  **Inicio del Servicio MySQL Router:** Inicia y habilita el servicio MySQL Router para que comience a enrutar conexiones.

### Fase 4: Configuración de la VM Cliente
1.  **Instalación de Software:** Instala un cliente MySQL y la herramienta Sysbench.
2.  **Prueba de Conexión:**
    *   Intenta conectarte desde la VM cliente al MySQL Router usando la IP de la VM Router y los puertos configurados (ej. `6446` para R/W, `6447` para R/O).
    *   Usa las credenciales del usuario de aplicación (ej. `cliente`) creado anteriormente.
    *   Verifica que las conexiones al puerto de lectura/escritura se dirigen al nodo primario del clúster y las conexiones al puerto de solo lectura se dirigen a los nodos secundarios (o al primario si es el único disponible).

### Fase 5: Ejecución de Pruebas
1.  **Validación de Permisos en Esclavos:** Confirma que los nodos esclavos están en modo solo lectura y no permiten operaciones de escritura directas.
2.  **Prueba de Alta Disponibilidad (Failover):**
    *   Simula una falla del nodo primario (ej. deteniendo el servicio MySQL en la VM Master).
    *   Observa (usando `cluster.status()` desde otro nodo) cómo el clúster detecta la falla y promueve automáticamente uno de los esclavos a nuevo primario.
    *   Verifica que el nuevo primario ahora acepta escrituras y que MySQL Router redirige las operaciones de escritura a este nuevo primario.
    *   Cuando el antiguo primario vuelva a estar en línea, debería reintegrarse al clúster como un secundario.
3.  **Pruebas de Carga con Sysbench:**
    *   **Preparación:** En la VM cliente, usa Sysbench para crear una base de datos de prueba (`sbtest`) y poblarla con datos, conectándote a través del puerto de lectura/escritura del MySQL Router.
    *   **Ejecución R/W:** Realiza una prueba de carga de lectura/escritura con Sysbench, apuntando al puerto de lectura/escritura del Router. Esto medirá el rendimiento del nodo primario.
    *   **Ejecución R/O:** Realiza una prueba de carga de solo lectura con Sysbench, apuntando al puerto de solo lectura del Router. Esto medirá el rendimiento combinado de los nodos secundarios (y posiblemente el primario, dependiendo de la configuración del router).
    *   **Limpieza:** Usa Sysbench para limpiar los datos de prueba.
