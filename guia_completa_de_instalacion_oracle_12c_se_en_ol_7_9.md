# GUÍA TÉCNICA DE INSTALACIÓN: ORACLE DATABASE 12c SE EN ORACLE LINUX 7U9

### Instalación Profesional en Modo Silencioso (Headless / Sin Interfaz Gráfica)
**Autor:** Alexander Machic  
**Versión de Base de Datos:** Oracle Database 12c Release 1 (12.1.0.2.0) Standard Edition (SE)  
**Sistema Operativo:** Oracle Linux 7 Update 9 (x86_64) con Unbreakable Enterprise Kernel (UEK) o RHCK  

---

## 1. Arquitectura de Referencia y Requisitos Previos

### 1.1. Especificaciones Mínimas de Hardware
* **RAM:** Mínimo 4 GB (8 GB o más recomendado para entornos productivos).
* **Swap:**
  * Entre 4 GB y 8 GB de RAM: Swap = 1x a 1.5x tamaño de RAM.
  * Entre 8 GB y 16 GB de RAM: Swap = tamaño de RAM.
  * Superior a 16 GB de RAM: 16 GB Swap fijo.
* **Espacio en Disco:**
  * `/u01` (Binarios de Software y Motor): Mínimo 30 GB.
  * `/u02` (Datos Principales / `oradata`): Mínimo 50 GB (según volumen proyectado).
  * `/u03` (Fast Recovery Area / Backups): Mínimo equivalente al doble del tamaño de los datos.

### 1.2. Arquitectura de Almacenamiento OFA y Segregación de Discos (Producción)

Tener `oradata` y la **Fast Recovery Area (FRA)** conviviendo en el mismo punto de montaje (`/u02`) representa un **riesgo crítico de disponibilidad y rendimiento**, especialmente en entornos de producción:

* **Punto único de fallo (SPOF - Single Point of Failure):** Si el volumen físico o LUN de `/u02` se corrompe o sufre una falla de hardware, se pierden simultáneamente los datos activos y las copias de seguridad / archivelogs necesarios para recuperarlos.
* **Riesgo de congelamiento de la base de datos (Database Hang):** Si la FRA se llena (por ejemplo, porque falló la ejecución o purga de backups de archivelogs con RMAN), ocupará el 100% de `/u02`. En ese instante, los datafiles no podrán autoextenderse y la base de datos se detendrá por completo (*hang* transaccional).
* **Contención de I/O (Cuello de botella transaccional):** Las lecturas y escrituras transaccionales aleatorias (OLTP) de los datafiles compiten directamente contra la escritura secuencial continua de los archivelogs y las lecturas/escrituras masivas de RMAN durante las ventanas de copia de seguridad.

#### Arquitectura Robusta Recomendada (Filesystem / OFA)
Una arquitectura empresarial estándar bajo la especificación **OFA (*Optimal Flexible Architecture*)** requiere separar el almacenamiento en al menos **tres puntos de montaje independientes**, respaldados por discos físicos, arreglos RAID o LUNs dedicadas distintas:

| Punto de montaje | Propósito principal | Tipo de I/O | Contenido |
| :--- | :--- | :--- | :--- |
| **`/u01`** | Binarios y Software | Lectura baja / Logs | `$ORACLE_HOME`, `$ORACLE_BASE`, inventario (`oraInventory`), trazas y `diag/`. |
| **`/u02`** | Datos Principales (DATA) | Aleatoria (OLTP) | Archivos de datos (`.dbf`), Miembro 1 de cada grupo de Redo Log, Copia 1 del Controlfile. |
| **`/u03`** | Área de Recuperación (FRA) | Secuencial intensiva | Archivelogs, Backups RMAN, Flashback logs, Miembro 2 de Redo Log, Copia 2 del Controlfile. |

> [!IMPORTANT]
> **Estrategia de Resiliencia y Multiplexación:**  
> Separar `/u02` y `/u03` en volúmenes físicamente independientes no solo aísla la carga de I/O y previene el congelamiento de la base de datos, sino que permite implementar la regla de oro de Oracle: **multiplexar los Controlfiles y los grupos de Redo Logs** entre volúmenes distintos. Si se pierde el almacenamiento de `/u02`, la base de datos cuenta con la copia del controlfile y el miembro del redo log en `/u03`, permitiendo la recuperación completa sin pérdida de información (*Zero Data Loss*).

### 1.3. Reglas de Licenciamiento para Oracle 12c R1 (12.1.0.2.0) Standard Edition (SE)
Para la versión **12.1.0.2.0**, las métricas y restricciones de licenciamiento oficiales son las siguientes:

* **Capacidad Máxima de Hardware (Sockets):**
  * Puede instalarse en servidores que cuenten con una capacidad física máxima de **hasta 4 sockets ocupados** (independientemente de la cantidad de cores o núcleos por procesador).
* **Métricas de Licenciamiento:**
  * **Processor License:** Se licencia por socket físico ocupado (sin aplicar la tabla de factor multincleo / *Core Processor Licensing Factor* de Enterprise Edition). Cada socket físico equivale a 1 licencia de procesador.
  * **Named User Plus (NUP):** Mínimo de **5 licencias NUP** por servidor u organización.
* **Uso de Recursos y Threads de CPU:**
  * En Standard Edition (SE) tradicional **no existe limitación artificial de threads de CPU** impuesta por software (a diferencia de SE2, el motor aprovecha la totalidad de threads y cores del hardware asignado).
* **Alta Disponibilidad con RAC (Real Application Clusters):**
  * Incluye soporte para Oracle RAC sin costo adicional de licenciamiento, siempre que la suma de sockets del clúster no supere los **4 sockets en total** (por ejemplo: 2 nodos con 2 sockets cada uno, o 4 nodos con 1 socket cada uno).
* **Arquitectura Multitenant (CDB / PDB):**
  * En 12.1.0.2 SE está permitida la arquitectura Container con un **máximo de 1 Pluggable Database (PDB)** de usuario (*Single-Tenant*) sin requerir la opción Multitenant de Enterprise Edition.

> **Nota sobre la transición histórica a SE2 en 12.1.0.2:**  
> En septiembre de 2015 (a partir del conjunto de parches 12.1.0.2), Oracle introdujo **Standard Edition 2 (SE2)** reemplazando a SE y SE1. Si el medio de instalación que utilizas corresponde específicamente al instalador de **12.1.0.2 SE2**, las reglas cambian a un **máximo de 2 sockets**, limitación por software de **16 CPU threads simultáneos** por instancia y un mínimo de **10 usuarios NUP** por servidor.

---

## 2. Preparación del Sistema Operativo (Como `root`)

### 2.1. Configuración de Red y Resolución de Nombres
Oracle requiere que el nombre del servidor resuelva a una dirección IP estática (no a `127.0.0.1` o DHCP).

1. Establecer el FQDN del servidor:
   ```bash
   hostnamectl set-hostname dbprod01.empresa.local
   ```

2. Editar `/etc/hosts` para mapear la IP estática:
   ```bash
   cat << 'EOF' >> /etc/hosts
   192.168.10.50   dbprod01.empresa.local   dbprod01
   EOF
   ```

3. Comprobar resolución:
   ```bash
   ping -c 2 $(hostname)
   ```

---

### 2.2. Instalación de Dependencias con el Paquete Pre-Install
Oracle Linux cuenta con un paquete oficial que configura de forma automática usuarios, grupos, límites de PAM y parámetros del kernel.

```bash
yum clean all
yum makecache
yum install -y oracle-database-server-12cR2-preinstall
```

Si el servidor no tiene salida a internet hacia el repositorio público de Oracle ULN / Yum, instala manualmente las dependencias mínimas:
```bash
yum install -y binutils compat-libcap1 compat-libstdc++-33 gcc gcc-c++ glibc \
glibc-devel ksh libaio libaio-devel libgcc libstdc++ libstdc++-devel \
libXi libXtst make sysstat smartmontools unzip xorg-x11-utils
```

---

### 2.3. Verificación y Ajuste de Parámetros del Kernel
Verifica el contenido de `/etc/sysctl.d/99-oracle-database-server-12cR2-preinstall.conf` (o crea `/etc/sysctl.d/99-oracle.conf` si no usaste el paquete RPM):

```ini
# Semáforos IPC: semmsl semmns semopm semmni
kernel.sem = 250 32000 100 128

# Memoria Compartida (Shared Memory)
kernel.shmmax = 12884901888   # Ejemplo: ~12 GB (ajustar a 75-80% de la RAM física)
kernel.shmall = 3145728       # Páginas (shmmax / 4096)
kernel.shmmni = 4096

# File Descriptors y Red
fs.file-max = 6815744
fs.aio-max-nr = 1048576
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
net.ipv4.ip_local_port_range = 9000 65500
```

Aplicar cambios en caliente:
```bash
sysctl --system
```

---

### 2.4. Grupos, Usuarios y Límites de Seguridad (`limits.conf`)
El paquete preinstall crea el usuario `oracle` y los grupos básicos. En entornos productivos, es buena práctica estructurar la separación de privilegios por roles de administración:

```bash
# Crear grupos complementarios si no existen
groupadd -g 54323 backupdba
groupadd -g 54324 dgdba
groupadd -g 54325 kmdba
groupadd -g 54326 racdba

# Modificar el usuario oracle
usermod -g oinstall -G dba,oper,backupdba,dgdba,kmdba,racdba oracle
passwd oracle
```

Validar `/etc/security/limits.d/oracle-database-server-12cR2-preinstall.conf` (debe contener):
```ini
oracle   soft   nofile    1024
oracle   hard   nofile    65536
oracle   soft   nproc     16384
oracle   hard   nproc     16384
oracle   soft   stack     10240
oracle   hard   stack     32768
oracle   hard   memlock   134217728
oracle   soft   memlock   134217728
```

---

### 2.5. Ajustes de Firewall y SELinux
Para una instalación estándar, habilita el puerto del Listener de Oracle (1521) en `firewalld`:

```bash
firewall-cmd --zone=public --add-port=1521/tcp --permanent
firewall-cmd --reload
```

Para **SELinux**, Oracle Database 12c está soportado con SELinux en modo `Enforcing`, pero si surgen conflictos de enlace con bibliotecas compartidas o drivers de almacenamiento, se recomienda situarlo en `Permissive`:
```bash
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config
setenforce 0
```

---

### 2.6. Deshabilitar Transparent HugePages (THP)
*Transparent HugePages* causa caídas de rendimiento, fragmentación de memoria y cuelgues inesperados en bases de datos Oracle.

1. Editar `/etc/default/grub` y añadir `transparent_hugepage=never` a la directiva `GRUB_CMDLINE_LINUX`:
   ```bash
   sed -i 's/GRUB_CMDLINE_LINUX="/GRUB_CMDLINE_LINUX="transparent_hugepage=never /' /etc/default/grub
   ```

2. Recompilar el cargador GRUB:
   ```bash
   grub2-mkconfig -o /boot/grub2/grub.cfg
   # En servidores UEFI usar:
   # grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
   ```

---

## 3. Estructura de Directorios (OFA) y Variables de Entorno

### 3.1. Creación de Directorios y Permisos
Seguiremos la convención OFA (*Optimal Flexible Architecture*) separando binarios, datos y área de recuperación:

```bash
# Directorios de software y binarios (/u01)
mkdir -p /u01/app/oracle/product/12.1.0/dbhome_1
mkdir -p /u01/app/oraInventory
mkdir -p /u01/stage

# Directorio de datos transaccionales / DATA (/u02)
mkdir -p /u02/oradata

# Directorio de Fast Recovery Area / FRA (/u03 recomendado en producción)
mkdir -p /u03/fast_recovery_area

# Asignar propiedad y permisos
chown -R oracle:oinstall /u01 /u02 /u03
chmod -R 775 /u01 /u02 /u03
```

> **Nota de Arquitectura (Producción vs Laboratorio):**  
> En cumplimiento con la especificación OFA para producción (Sección 1.2), `/u03` corresponde al punto de montaje en un disco físico o LUN independiente para la FRA. Si te encuentras en un entorno de desarrollo o laboratorio con un único disco de datos disponible, puedes utilizar provisionalmente `/u02/fast_recovery_area` (`mkdir -p /u02/fast_recovery_area`) y omitir `/u03`.

---

### 3.2. Configuración del `.bash_profile` de `oracle`
Inicia sesión como el usuario `oracle`:
```bash
su - oracle
```

Edita `/home/oracle/.bash_profile` añadiendo el siguiente bloque:

```bash
cat << 'EOF' >> ~/.bash_profile

# --------------------------------------------------
# Entorno de Producción Oracle 12c SE
# --------------------------------------------------
export ORACLE_HOSTNAME=dbprod01.empresa.local
export ORACLE_UNQNAME=PROD12C
export ORACLE_SID=PROD12C
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/12.1.0/dbhome_1
export ORA_INVENTORY=/u01/app/oraInventory

export TNS_ADMIN=$ORACLE_HOME/network/admin
export NLS_LANG=SPANISH_SPAIN.AL32UTF8
export NLS_DATE_FORMAT="YYYY-MM-DD HH24:MI:SS"

export PATH=/usr/sbin:/usr/local/bin:$PATH
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib

umask 022
EOF

source ~/.bash_profile
```

---

## 4. Descompresión de Binarios

Copia el medio de instalación oficial de Oracle 12.1.0.2.0 (ej. `linuxx64_12201_database.zip`) en el directorio `/u01/stage` y descomprímelo como usuario `oracle`:

```bash
cd /u01/stage
unzip -q linuxx64_12201_database.zip
```

Esto generará la carpeta `/u01/stage/database`.

---

## 5. Instalación del Software en Modo Silencioso (*runInstaller*)

### 5.1. Creación del Archivo de Respuesta `db_install.rsp`
Como usuario `oracle`, crea el archivo `/u01/stage/db_install_se2.rsp`:

```ini
# ==============================================================================
# ARCHIVO DE RESPUESTA PARA INSTALACIÓN SILENCIOSA DE ORACLE 12.1 SE
# ==============================================================================
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v12.1.0
oracle.install.option=INSTALL_DB_SWONLY
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/u01/app/oraInventory
ORACLE_HOME=/u01/app/oracle/product/12.1.0/dbhome_1
ORACLE_BASE=/u01/app/oracle

# DEFINICIÓN ESPECÍFICA DE EDICIÓN: Standard Edition 2
oracle.install.db.InstallEdition=SE2

# Grupos de seguridad de la base de datos (Sintaxis 12.1.0.2.0)
oracle.install.db.DBA_GROUP=dba
oracle.install.db.OPER_GROUP=oper
oracle.install.db.BACKUPDBA_GROUP=backupdba
oracle.install.db.DGDBA_GROUP=dgdba
oracle.install.db.KMDBA_GROUP=kmdba

# Desactivar soporte de Oracle Support / My Oracle Support en la instalación
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false
DECLINE_SECURITY_UPDATES=true
```

---

### 5.2. Ejecución del Instalador en Modo Silencioso
Ejecuta `runInstaller` pasando el archivo de respuesta:

```bash
cd /u01/stage/database

./runInstaller -silent \
  -ignorePrereq \
  -waitForCompletion \
  -responseFile /u01/stage/db_install_se2.rsp
```

*Monitoreo:* El instalador emitirá la ruta del log en pantalla (habitualmente bajo `/u01/app/oraInventory/logs/installActions<fecha>.log`). Puedes abrir otra terminal y seguirlo con `tail -f`.

---

### 5.3. Ejecución de los Scripts Raíz (Como `root`)
Al concluir la copia y compilación de binarios, el instalador indicará la necesidad de ejecutar dos scripts como superusuario.

Abre una terminal como `root` y ejecútalos en orden:

```bash
# 1. Script de inventario
/u01/app/oraInventory/orainstRoot.sh

# 2. Script de configuración global de Oracle Home y oratab
/u01/app/oracle/product/12.1.0/dbhome_1/root.sh
```

---

## 6. Configuración del Listener en Modo Silencioso (NetCA)

Como usuario `oracle`, utiliza `netca` en modo silencioso para levantar el Listener predeterminado en el puerto 1521:

```bash
su - oracle
netca -silent -responsefile $ORACLE_HOME/assistants/netca/netca.rsp
```

Verifica el estado del servicio de escucha:
```bash
lsnrctl status
```
*Deberás observar el Listener `LISTENER` iniciado escuchando en el puerto 1521 en estado UP.*

---

## 7. Creación de la Base de Datos con DBCA (Modo Silencioso)

Para bases de datos de producción se recomienda separar datafiles de la Fast Recovery Area, usar el juego de caracteres universal `AL32UTF8` y dimensionar los Redo Logs adecuadamente para evitar frecuentes *log file switch checkpoints incomplete*.

### 7.1. Ejecución de DBCA
Ejecuta el siguiente comando como usuario `oracle`:

```bash
dbca -silent -createDatabase \
  -templateName General_Purpose.dbc \
  -gdbName PROD12C.empresa.local \
  -createAsContainerDatabase true \
  -pdbName PDB1 \
  -emConfiguration NONE \
  -datafileDestination /u02/oradata \
  -storageType FS \
  -characterSet AL32UTF8 \
  -nationalCharacterSet AL16UTF16 \
  -sampleSchema false \
  -memoryPercentage 40 \
  -automaticMemoryManagement false \
  -initParams open_cursors=800,processes=500 \
  -sysPassword 'manager1' \
  -systemPassword 'manager1'
```

> **Nota sobre Arquitectura Multitenant en 12.1.0.2.0 SE:**  
> En **Oracle Database 12c Release 1 (12.1.0.2.0) Standard Edition (SE)**, la arquitectura Multitenant está permitida bajo la modalidad **Single-Tenant**, que incluye exactamente **1 Pluggable Database (PDB)** de usuario sin costo adicional de licenciamiento.  
> * **Para mantener la arquitectura CDB con 1 PDB (Recomendado):** Utiliza el comando tal como está especificado arriba (`-createAsContainerDatabase true -pdbName PDB1`).  
> * **Para crear una base de datos tradicional (Non-CDB):** En Oracle 12.1.0.2 la arquitectura clásica Non-CDB aún está completamente disponible; si prefieres desplegar una Non-CDB, sustituye `-createAsContainerDatabase true -pdbName PDB1` por `-createAsContainerDatabase false` (omitiendo la opción `-pdbName`).  
> * **Aprovisionamiento manual:** Si DBCA inicializa el contenedor (`CDB$ROOT` + `PDB$SEED`) sin crear la PDB de usuario automáticamente, el aprovisionamiento de `PDB1` se realiza de manera directa en el paso 8.3 tras configurar la FRA y el modo Archivelog.

---

## 8. Optimizaciones Críticas Post-Instalación

Conéctate a la base de datos como usuario `oracle` vía SQL*Plus:
```bash
su - oracle
sqlplus / as sysdba
```

### 8.1. Configuración y Activación Manual de la Fast Recovery Area (FRA)
La *Fast Recovery Area* (FRA) es la ubicación de almacenamiento centralizada y administrada por Oracle (OMF) para alojar respaldos de RMAN, copias de controlfiles y los registros de transacciones archivados (*Archived Redo Logs*).

Al haberse creado la base de datos en el punto 7 sin definir los parámetros de FRA en DBCA, **es obligatorio configurarla antes de habilitar el modo ARCHIVELOG**. De lo contrario, Oracle generará los archivelogs en el destino por defecto del motor (`$ORACLE_HOME/dbs`), saturando el volumen de binarios `/u01`.

#### 8.1.1. Comprobación del Directorio Físico
Verifica que el directorio `/u03/fast_recovery_area` (o `/u02/fast_recovery_area` si se utiliza un único disco en laboratorio) exista y mantenga permisos de escritura para `oracle:oinstall`:
```bash
ls -ld /u03/fast_recovery_area
# Debe mostrar propietario oracle:oinstall y permisos 775 (drwxrwxr-x)
# (En entornos de laboratorio: ls -ld /u02/fast_recovery_area)
```

#### 8.1.2. Configuración de Parámetros Dinámicos (SQL*Plus)
Dentro de la sesión `sqlplus / as sysdba`, ejecuta los siguientes comandos:

> **Regla Crítica de Dependencia:** Oracle exige definir `DB_RECOVERY_FILE_DEST_SIZE` **antes** que `DB_RECOVERY_FILE_DEST`. Si se asigna la ruta sin un tamaño previo, el motor retornará el error:  
> `ORA-01261: Parameter db_recovery_file_dest destination string cannot be set to '' without db_recovery_file_dest_size`.

```sql
-- 1. Definir la cuota máxima de espacio para la FRA (ajustar según el disco /u03, ej. 50 GB)
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST_SIZE = 50G SCOPE=BOTH;

-- 2. Definir la ruta física de la FRA (recomendado /u03 en volumen independiente; o /u02 para lab)
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST = '/u03/fast_recovery_area' SCOPE=BOTH;

-- 3. Enlazar explícitamente el destino principal de archivado a la FRA
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1 = 'LOCATION=USE_DB_RECOVERY_FILE_DEST' SCOPE=BOTH;

-- 4. Establecer el formato estándar de nombrado de Redo Logs Archivados
ALTER SYSTEM SET LOG_ARCHIVE_FORMAT = '%t_%s_%r.arc' SCOPE=SPFILE;
```

#### 8.1.3. Validación de Parámetros y Cuota de la FRA
Comprueba que los cambios estén activos en memoria y en el archivo de parámetros (SPFILE):
```sql
-- Verificar parámetros activos
SHOW PARAMETER db_recovery;

-- Consultar espacio total, espacio en uso y archivos alojados en la FRA
SELECT name, 
       ROUND(space_limit / 1024 / 1024 / 1024, 2) AS limit_gb, 
       ROUND(space_used / 1024 / 1024 / 1024, 2)  AS used_gb, 
       ROUND(space_reclaimable / 1024 / 1024 / 1024, 2) AS recl_gb, 
       number_of_files 
FROM v$recovery_file_dest;
```

---

### 8.2. Habilitación de Modo ARCHIVELOG
Con la FRA configurada y verificada, se procede a habilitar el modo ARCHIVELOG. Para cambiar el modo de registro de la base de datos, la instancia debe montarse sin abrirse (`MOUNT`).

#### 8.2.1. Conmutación a Modo ARCHIVELOG
```sql
-- Bajar la instancia de forma consistente
SHUTDOWN IMMEDIATE;

-- Iniciar en modo MOUNT
STARTUP MOUNT;

-- Habilitar el modo de archivado
ALTER DATABASE ARCHIVELOG;

-- Abrir la base de datos
ALTER DATABASE OPEN;
```

#### 8.2.2. Validación y Prueba de Generación en Disco
```sql
-- Validar estado del archivador
ARCHIVE LOG LIST;
SELECT name, log_mode, open_mode FROM v$database;

-- Forzar un switch de log para certificar la escritura física en la FRA
ALTER SYSTEM CHECKPOINT;
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM ARCHIVE LOG CURRENT;

-- Verificar el último archivelog generado y su ruta en la FRA
SELECT sequence#, name, status, completion_time 
FROM v$archived_log 
WHERE sequence# = (SELECT MAX(sequence#) FROM v$archived_log);
```

Desde la terminal del sistema operativo, corrobora la creación de la estructura OMF bajo la FRA:
```bash
ls -la /u03/fast_recovery_area/PROD12C/archivelog/
# (O /u02/fast_recovery_area/PROD12C/archivelog/ en caso de laboratorio)
```

---

### 8.3. Creación y Apertura Persistente de la Pluggable Database (PDB1)
En la arquitectura Multitenant de Oracle 12c, una vez configurada la FRA y habilitado el modo ARCHIVELOG en la base de datos raíz (`CDB$ROOT`), se procede al aprovisionamiento y configuración de la base de datos conectable de usuario (`PDB1`).

#### 8.3.1. Validación Previa del Entorno Multitenant
Antes de proceder con la creación de la PDB, valida que la instancia sea efectivamente un contenedor (CDB), que la sesión se encuentre ubicada en la raíz (`CDB$ROOT`) y que aún no exista la PDB de usuario:

```sql
-- 1. ¿La base de datos es un contenedor (CDB) o es tradicional (Non-CDB)?
SELECT name, cdb, open_mode FROM v$database;

-- 2. ¿En qué contenedor estás ubicado actualmente?
SHOW CON_NAME;

-- 3. ¿Qué registros existen en v$pdbs?
SELECT con_id, name, open_mode FROM v$pdbs;
```

*Si `CDB` devuelve `YES`, `CON_NAME` indica `CDB$ROOT` y `v$pdbs` únicamente lista `PDB$SEED` en `READ ONLY`, confirma que la base de datos es un contenedor y está lista para aprovisionar `PDB1`.*

#### 8.3.2. Creación y Configuración de `PDB1`
Conéctate a la base de datos como `sysdba` en `CDB$ROOT` y ejecuta los siguientes comandos para crear `PDB1` clonándola desde la plantilla del sistema (`PDB$SEED`):

```sql
-- 1. Crear la Pluggable Database PDB1
CREATE PLUGGABLE DATABASE PDB1 
  ADMIN USER pdbadmin IDENTIFIED BY manager1 
  CREATE_FILE_DEST = '/u02/oradata';

-- 2. Abrir la nueva PDB en modo lectura y escritura (READ WRITE)
ALTER PLUGGABLE DATABASE PDB1 OPEN READ WRITE;

-- 3. Guardar el estado para que inicie automáticamente en cada arranque
ALTER PLUGGABLE DATABASE PDB1 SAVE STATE;

-- 4. Validar el resultado
SELECT con_id, name, open_mode, restricted FROM v$pdbs;
```

#### Salida esperada en la consulta:
```text
    CON_ID NAME       OPEN_MODE  RESTRICTED
---------- ---------- ---------- ----------
         2 PDB$SEED   READ ONLY  NO
         3 PDB1       READ WRITE NO
```

> **Aclaración sobre `READ ONLY` en `v$pdbs`:**
> * **`PDB$SEED` (CON_ID = 2):** Es la plantilla de solo lectura del sistema utilizada por el motor Oracle para aprovisionar nuevas PDBs. Por arquitectura interna, **SIEMPRE debe estar en modo `READ ONLY`** (nunca puede estar en `READ WRITE`). Este comportamiento es completamente normal y correcto.
> * **`PDB1` (CON_ID = 3):** Es la base de datos productiva de usuario y **es la única que debe estar en `READ WRITE`**.
>
> *(Resolución de incidencias)* Si `PDB1` ya existía previamente pero figurara en `READ ONLY`, fuerce su reapertura explícita ejecutando:
> ```sql
> ALTER PLUGGABLE DATABASE PDB1 CLOSE IMMEDIATE;
> ALTER PLUGGABLE DATABASE PDB1 OPEN READ WRITE;
> ALTER PLUGGABLE DATABASE PDB1 SAVE STATE;
> ```
> Si arrojara algún error al intentar abrir en `READ WRITE`, verifique si existen violaciones no resueltas en el catálogo:
> ```sql
> SELECT name, cause, message, status FROM pdb_plug_in_violations WHERE status != 'RESOLVED';
> ```

---

### 8.4. Redimensionamiento y Multiplexación de Redo Logs
La plantilla de propósito general suele crear 3 grupos de solo 200 MB con un único miembro, lo cual genera saturación de I/O en producción y riesgo de pérdida de datos ante fallo de disco. Se deben crear nuevos grupos de mínimo 1 GB (1024 MB) y multiplexar sus miembros entre `/u02` (DATA) y `/u03` (FRA) conforme a la arquitectura OFA definida en la Sección 1.2:

```sql
-- Consultar grupos actuales
SELECT group#, bytes/1024/1024 MB, status FROM v$log;

-- Crear nuevos grupos de 1 GB multiplexados entre /u02 (DATA) y /u03 (FRA)
ALTER DATABASE ADD LOGFILE GROUP 4 (
  '/u02/oradata/PROD12C/redo04a.log',
  '/u03/fast_recovery_area/PROD12C/redo04b.log'
) SIZE 1024M;

ALTER DATABASE ADD LOGFILE GROUP 5 (
  '/u02/oradata/PROD12C/redo05a.log',
  '/u03/fast_recovery_area/PROD12C/redo05b.log'
) SIZE 1024M;

ALTER DATABASE ADD LOGFILE GROUP 6 (
  '/u02/oradata/PROD12C/redo06a.log',
  '/u03/fast_recovery_area/PROD12C/redo06b.log'
) SIZE 1024M;

-- Conmutar logs hasta que los grupos viejos (1, 2, 3) queden en estado INACTIVE
ALTER SYSTEM CHECKPOINT;
ALTER SYSTEM SWITCH LOGFILE;

-- Eliminar los grupos viejos pequeños (1, 2, 3) una vez inactivos
ALTER DATABASE DROP LOGFILE GROUP 1;
ALTER DATABASE DROP LOGFILE GROUP 2;
ALTER DATABASE DROP LOGFILE GROUP 3;
```

> [!TIP]
> **Multiplexación de Controlfiles y Redo Logs en Laboratorios:**  
> En producción, esta configuración asegura que ante la caída total del almacenamiento `/u02` o `/u03`, la base de datos mantenga un miembro íntegro de redo log activo. Si ejecutas esta guía en una máquina virtual de desarrollo con un solo disco adicional, puedes alojar el segundo miembro bajo `/u02/fast_recovery_area/PROD12C/redo0Xb.log`.  
> Del mismo modo, valida que el parámetro `CONTROL_FILES` apunte a copias distribuidas (ej. `/u02/oradata/PROD12C/control01.ctl` y `/u03/fast_recovery_area/PROD12C/control02.ctl`).

---

### 8.5. Configuración de Retención de Metadatos en Controlfile
Define la cantidad mínima de días que se retienen los metadatos de respaldos RMAN y registros de archivelogs en el controlfile antes de permitir su sobreescritura (por defecto son 7 días):

```sql
-- Configurar retención a 30 días
ALTER SYSTEM SET CONTROL_FILE_RECORD_KEEP_TIME = 30 SCOPE=BOTH;

-- Validar parámetro
SHOW PARAMETER control_file_record_keep_time;
```

---

## 9. Automatización de Inicio y Parada del Sistema (Systemd)

En Oracle Linux 7, la gestión de servicios se realiza mediante `systemd`. Configuraremos un servicio nativo para iniciar y detener el Listener y la Base de Datos con el arranque del sistema.

### 9.1. Habilitar la Base de Datos en `/etc/oratab` (Como `root`)
Modifica `/etc/oratab` cambiando la bandera final de `N` a `Y`:

```bash
# Cambiar:
# PROD12C:/u01/app/oracle/product/12.1.0/dbhome_1:N
# Por:
sed -i 's/PROD12C:\/u01\/app\/oracle\/product\/12.1.0\/dbhome_1:N/PROD12C:\/u01\/app\/oracle\/product\/12.1.0\/dbhome_1:Y/g' /etc/oratab
```

---

### 9.2. Creación del Servicio Systemd
Crea el archivo `/etc/systemd/system/oracle-rdbms.service` como `root`:

```ini
[Unit]
Description=Oracle Database 12c Services
After=syslog.target network.target

[Service]
Type=forking
User=oracle
Group=oinstall
RemainAfterExit=yes
KillMode=none

# Variables de entorno mínimas para los scripts
Environment="ORACLE_HOME=/u01/app/oracle/product/12.1.0/dbhome_1"
Environment="ORACLE_BASE=/u01/app/oracle"

# Comandos de Inicio
ExecStart=/u01/app/oracle/product/12.1.0/dbhome_1/bin/dbstart /u01/app/oracle/product/12.1.0/dbhome_1

# Comandos de Parada
ExecStop=/u01/app/oracle/product/12.1.0/dbhome_1/bin/dbshut /u01/app/oracle/product/12.1.0/dbhome_1

[Install]
WantedBy=multi-user.target
```

Habilita y arranca el servicio:
```bash
systemctl daemon-reload
systemctl enable oracle-rdbms.service
```

---

## 10. Checklist de Verificación Final

Ejecuta las siguientes comprobaciones para certificar que el entorno está listo para producción:

1. **Estado del Listener:**
   ```bash
   su - oracle -c "lsnrctl status"
   ```
   *El servicio `PROD12C` debe figurar como `READY`.*

2. **Acceso SQL y Versión:**
   ```bash
   su - oracle -c "sqlplus -s / as sysdba <<< 'SELECT banner FROM v\$version;'"
   ```
   *Debe confirmar `Oracle Database 12c ... Standard Edition Release 12.1.0.2.0`.*

3. **Estado de la Fast Recovery Area (FRA):**
   ```bash
   su - oracle -c "sqlplus -s / as sysdba <<< 'SELECT name, space_limit/1024/1024/1024 GB_LIMIT, space_used/1024/1024/1024 GB_USED FROM v\$recovery_file_dest;'"
   ```
   *Debe listar `/u03/fast_recovery_area` (o `/u02/fast_recovery_area` en laboratorios) y mostrar el espacio configurado.*

4. **Modo Archivelog:**
   ```bash
   su - oracle -c "sqlplus -s / as sysdba <<< 'SELECT log_mode FROM v\$database;'"
   ```
   *Debe responder `ARCHIVELOG`.*

5. **Estado de la Pluggable Database (`PDB1`):**
   ```bash
   su - oracle -c "sqlplus -s / as sysdba <<< 'SELECT name, open_mode FROM v\$pdbs WHERE name = '\''PDB1'\'';'"
   ```
   *Debe responder `READ WRITE`.*

6. **Prueba de Reinicio del Sistema:**
   ```bash
   reboot
   ```
   *Tras el reinicio del servidor, comprueba que la instancia `PROD12C`, el Listener y la `PDB1` hayan levantado de forma autónoma sin intervención humana.*
