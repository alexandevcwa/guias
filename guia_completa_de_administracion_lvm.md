# GUÍA TÉCNICA DE ADMINISTRACIÓN DE LVM (LOGICAL VOLUME MANAGER)

### Manual de Operaciones y Mantenimiento para Oracle Linux 7 / Red Hat Enterprise Linux 7

## 1. Conceptos Fundamentales y Arquitectura

LVM desacopla el almacenamiento físico del sistema de archivos, permitiendo redimensionar, añadir o migrar particiones sin reiniciar el servidor.

### 1.1. Capas de Abstracción

```
+--------------------------------------------------------------------------+
|  Puntos de Montaje:   /          /u01         /u02         /var    /tmp  |
|  Sistemas de Archivo: (XFS)      (XFS)        (XFS/ext4)   (XFS)   (XFS) |
+--------------------------------------------------------------------------+
                                    ^
+--------------------------------------------------------------------------+
|  Logical Volumes (LV):                                                  |
|  lv_root          lv_u01        lv_u02       lv_var        lv_tmp        |
+--------------------------------------------------------------------------+
                                    ^
+--------------------------------------------------------------------------+
|  Volume Group (VG):                                                      |
|  vg_system (o vg_oracle)                                                 |
|  Pool global de espacio dividido en Physical Extents (PE, default: 4MB)  |
+--------------------------------------------------------------------------+
                                    ^
+--------------------------------------------------------------------------+
|  Physical Volumes (PV):                                                  |
|  /dev/sda2              /dev/sdb1               /dev/sdc                 |
+--------------------------------------------------------------------------+
                                    ^
+--------------------------------------------------------------------------+
|  Discos Físicos / LUNs SAN / Discos Virtuales (sda, sdb, sdc...)         |
+--------------------------------------------------------------------------+

```

* **PV (Physical Volume):** Disco crudo o partición (`/dev/sdb`, `/dev/nvme0n1p1`) inicializado para que LVM lo gestione.

* **VG (Volume Group):** Grupo de almacenamiento que reúne uno o más PVs en un solo depósito (*pool*).

* **LV (Logical Volume):** Dispositivo de bloque virtual creado dentro del VG. Es el equivalente a una partición tradicional donde se monta el sistema de archivos (`mkfs`).

* **PE (Physical Extent):** La unidad mínima de asignación dentro de un VG (generalmente 4 MB).

## 2. Diagnóstico e Inspección del Almacenamiento

Antes de cualquier alteración en producción, debes inspeccionar la topología actual.

### 2.1. Comandos Rápidos de Resumen

```
# Ver topología de bloques, puntos de montaje y tamaños
lsblk -f

# Resumen de PVs, VGs y LVs en formato compacto
pvs
vgs
lvs

```

### 2.2. Comandos Detallados

```
# Espacio libre y detalle del Volume Group (identifica Free PE / Size)
vgdisplay

# Detalle de volúmenes lógicos y rutas completas
lvdisplay

# Identificar tipo de filesystem de cada partición (XFS vs ext4)
df -Th

```

> **REGLA CRÍTICA DE ORO (XFS vs ext4):**
> En Oracle Linux 7, el sistema de archivos por defecto es **XFS**.
>
> * **XFS admite expansión online (crecer), pero NO ADMITE REDUCCIÓN bajo ninguna circunstancia.**
>
> * Si un volumen es XFS y lo reduces con LVM, corromperás irremediablemente el filesystem.
>
> * **ext4** admite reducción, pero requiere desmontar el volumen (`umount`) previamente.

## 3. Ampliación de Espacio (Casos Más Frecuentes)

### Escenario 3.1: Se amplió el disco virtual existente en el hipervisor (VMware/KVM/SAN)

*Ejemplo:* El disco `/dev/sda` pasó de 100 GB a 150 GB en caliente.

1. **Escanear el bus SCSI para detectar el nuevo tamaño sin reiniciar:**

   ```
   for host in /sys/class/scsi_host/host*/scan; do echo "- - -" > $host; done
   echo 1 > /sys/class/block/sda/device/rescan
   
   ```

   *Verificar con `lsblk` que `/dev/sda` muestra el nuevo tamaño.*

2. **Ajustar la partición (si aplica, ej. `/dev/sda2`):**

   ```
   # Instalar cloud-utils-growpart si no está disponible
   yum install -y cloud-utils-growpart
   growpart /dev/sda 2
   
   ```

3. **Redimensionar el Physical Volume (PV):**

   ```
   pvresize /dev/sda2
   pvs   # Verás que PFree (espacio libre) aumentó
   
   ```

4. **Extender el Logical Volume y el Filesystem:**

   ```
   # El parámetro -r (--resizefs) ajusta automáticamente XFS o ext4 al mismo tiempo
   lvextend -r -L +40G /dev/mapper/vg_system-lv_u02
   
   ```

### Escenario 3.2: Se añadió un disco físico nuevo al servidor (`/dev/sdb`)

*Ejemplo:* Queremos añadir 200 GB para darle espacio a `/u02` (archivos de Oracle).

1. **Detectar el nuevo disco:**

   ```
   lsblk
   # Supongamos que el nuevo disco no particionado es /dev/sdb
   
   ```

2. **Inicializar como Physical Volume (PV):**

   ```
   pvcreate /dev/sdb
   
   ```

3. **Añadir el PV al Volume Group existente (identifica el nombre con `vgs`):**

   ```
   # Supongamos que tu VG se llama vg_system
   vgextend vg_system /dev/sdb
   vgs   # Comprueba que VFree ahora incluye los 200G nuevos
   
   ```

4. **Extender el volumen lógico deseado:**

   ```
   # Opción A: Añadir una cantidad fija (+100 GB)
   lvextend -r -L +100G /dev/vg_system/lv_u02
   
   # Opción B: Asignar el 100% de todo el espacio libre que queda en el VG
   lvextend -r -l +100%FREE /dev/vg_system/lv_u02
   
   ```

### 3.3. Expansión manual del Filesystem (si no usaste la opción `-r`)

Si ejecutaste `lvextend` sin `-r`, debes expandir la capa del sistema de archivos manualmente:

* **Para XFS (estándar en Oracle Linux 7):**

  ```
  # Requiere la ruta del punto de montaje, NO del dispositivo:
  xfs_growfs /u02
  
  ```

* **Para ext4:**

  ```
  # Requiere la ruta del dispositivo:
  resize2fs /dev/mapper/vg_system-lv_u02
  
  ```

## 4. Creación de Nuevos Volúmenes Lógicos (Ej. `/u03` para Backups)

Supongamos que dispones de espacio libre en el VG `vg_system` y necesitas crear un punto de montaje `/u03`.

### Paso 4.1: Crear el Logical Volume

```
# Crear un volumen de 80 GB llamado lv_u03
lvcreate -L 80G -n lv_u03 vg_system

```

### Paso 4.2: Dar formato al volumen

*Para XFS (Recomendado en OL7):*

```
mkfs.xfs -f /dev/vg_system/lv_u03

```

*Para ext4 (Alternativa):*

```
mkfs.ext4 /dev/vg_system/lv_u03

```

### Paso 4.3: Crear directorio y montar

```
mkdir -p /u03
mount /dev/vg_system/lv_u03 /u03

```

### Paso 4.4: Registrar en `/etc/fstab` de forma persistente

Obtén el UUID del nuevo dispositivo:

```
blkid /dev/vg_system/lv_u03

```

*Salida:* `/dev/mapper/vg_system-lv_u03: UUID="3f1a6c8e-28b3-4f91-8ad2-123456789abc" TYPE="xfs"`

Edita `/etc/fstab` y agrega la línea correspondiente:

```
UUID=3f1a6c8e-28b3-4f91-8ad2-123456789abc  /u03  xfs  defaults,noatime,_netdev  0 0

```

Valida la sintaxis de `/etc/fstab` para evitar problemas de arranque:

```
# Desmontar y probar el montaje mediante fstab
umount /u03
mount -a
df -h /u03

```

## 5. Reducción y Eliminación de Volúmenes Lógicos

### 5.1. Eliminación Completa de un Logical Volume (Destructivo)

*Ejemplo:* El volumen `/u03` ya no se utiliza y queremos devolver ese espacio al VG.

1. **Desmontar el volumen:**

   ```
   umount /u03
   
   ```

   *Si da error de busy ("target is busy"), identifica y mata los procesos:*

   ```
   fuser -mv /u03
   fuser -kv /u03
   umount /u03
   
   ```

2. **Eliminar la línea en `/etc/fstab`:**
   Borra o comenta la línea de `/u03` en `/etc/fstab`.

3. **Eliminar el Logical Volume:**

   ```
   lvremove /dev/vg_system/lv_u03
   # Confirmar con 'y'
   
   ```

   *El espacio queda automáticamente libre en el VG.*

### 5.2. Reducción de un Logical Volume (SOLO ext4)

> ⚠️ **PELIGRO:** **No intentes esto en volúmenes XFS.** Si el volumen es XFS, el único camino para reducirlo es hacer un respaldo completo de los datos (tar/cpio/rman), eliminar el LV, recrearlo más pequeño y restaurar la información.

Para volúmenes con formato **ext4**:

1. **Detener servicios (Oracle) y desmontar:**

   ```
   umount /u02
   
   ```

2. **Obligatorio: Chequeo de integridad del filesystem:**

   ```
   e2fsck -f /dev/vg_system/lv_u02
   
   ```

3. **Reducir PRIMERO el sistema de archivos (ejemplo: a 50 GB):**

   ```
   resize2fs /dev/vg_system/lv_u02 50G
   
   ```

4. **Reducir SEGUNDO el Logical Volume al mismo tamaño:**

   ```
   lvreduce -L 50G /dev/vg_system/lv_u02
   
   ```

5. **Volver a montar y validar:**

   ```
   mount /dev/vg_system/lv_u02 /u02
   df -h /u02
   
   ```

## 6. Retirar y Reemplazar Discos de un Volume Group

*Escenario:* Tienes dos discos en el VG (`/dev/sda2` y `/dev/sdb`). El disco `/dev/sdb` presenta fallas de hardware y debe ser reemplazado sin detener el sistema.

1. **Añadir el disco nuevo de reemplazo (`/dev/sdc`):**

   ```
   pvcreate /dev/sdc
   vgextend vg_system /dev/sdc
   
   ```

2. **Mover en caliente todos los bloques ocupados desde el disco dañado al nuevo:**

   ```
   # pvmove migra los Physical Extents transparentemente sin desmontar
   pvmove /dev/sdb /dev/sdc
   
   ```

3. **Retirar el disco dañado del Volume Group:**

   ```
   vgreduce vg_system /dev/sdb
   
   ```

4. **Desactivar el PV para extracción física segura:**

   ```
   pvremove /dev/sdb
   
   ```

## 7. Snapshots con LVM (Copias Consistentes)

Un Snapshot LVM crea una imagen en un instante de tiempo mediante *Copy-on-Write* (CoW). Requiere espacio libre no asignado en el VG.

### 7.1. Tomar un Snapshot Consistente de Oracle

Para que el snapshot sea consistente con la base de datos:

1. **Poner la BD en modo backup (vía SQL\*Plus):**

   ```
   ALTER DATABASE BEGIN BACKUP;
   
   ```

2. **Crear el snapshot LVM (ejemplo: reservando 20 GB para los cambios):**

   ```
   lvcreate -L 20G -s -n lv_u02_snap /dev/vg_system/lv_u02
   
   ```

3. **Finalizar el modo backup en Oracle:**

   ```
   ALTER DATABASE END BACKUP;
   
   ```

### 7.2. Montar el Snapshot (Especial para XFS)

XFS no permite montar dos particiones con el mismo UUID al mismo tiempo en el mismo host. Se debe usar el argumento `nouuid`:

```
mkdir -p /mnt/snapshot_u02
mount -o nouuid,ro /dev/vg_system/lv_u02_snap /mnt/snapshot_u02

```

### 7.3. Eliminar el Snapshot una vez utilizado

```
umount /mnt/snapshot_u02
lvremove -y /dev/vg_system/lv_u02_snap

```

## 8. Guía de Respuestas a Incidentes Frecuentes

### Incidente 1: `/tmp` o `/var` al 100% (Base de Datos o SO Bloqueado)

1. Verificar espacio libre en el VG:

   ```
   vgs
   
   ```

2. Si hay espacio libre (`VFree > 5G`), expandir en caliente de inmediato:

   ```
   lvextend -r -L +5G /dev/mapper/vg_system-lv_tmp
   lvextend -r -L +10G /dev/mapper/vg_system-lv_var
   
   ```

### Incidente 2: Servidor no arranca por error en `/etc/fstab`

Si modificaste `/etc/fstab` incorrectamente, el sistema caerá en modo `Emergency Mode`.

1. Ingresar la contraseña de `root`.

2. Remontar la raíz con permisos de escritura:

   ```
   mount -o remount,rw /
   
   ```

3. Editar `/etc/fstab`, corregir o comentar las líneas problemáticas:

   ```
   vi /etc/fstab
   
   ```

4. Reiniciar:

   ```
   systemctl reboot
   
   ```

## 9. Hoja de Atajos y Referencia Rápida

| Acción | Comando | 
| ----- | ----- | 
| **Ver espacio libre en VG** | `vgs -o +vg_free` | 
| **Ver en qué disco físico está cada LV** | `lvs -a -o +devices` | 
| **Crear PV** | `pvcreate /dev/sdb` | 
| **Añadir disco a VG** | `vgextend vg_system /dev/sdb` | 
| **Extender LV + Filesystem (+20 GB)** | `lvextend -r -L +20G /dev/vg_system/lv_u01` | 
| **Extender LV usando todo el VG** | `lvextend -r -l +100%FREE /dev/vg_system/lv_u02` | 
| **Crear LV de 50 GB** | `lvcreate -L 50G -n lv_nuevo vg_system` | 
| **Formatear XFS** | `mkfs.xfs /dev/vg_system/lv_nuevo` | 
| **Crear Snapshot de 10 GB** | `lvcreate -L 10G -s -n lv_snap /dev/vg_system/lv_u02` | 
| **Desconectar disco de VG** | `pvmove /dev/sdb && vgreduce vg_system /dev/sdb` | 
| **Eliminar LV** | `lvremove /dev/vg_system/lv_obsoleto` | 
