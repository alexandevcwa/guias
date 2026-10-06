# Guía Técnica de Optimización de Máquina Virtual Windows en QEMU/KVM

**Sistema Anfitrión (Host):** Manjaro Linux (Kernel 6.18)  
**Procesador Host:** Intel Core i5-13420H (12 hilos: 4 P-Cores con Hyper-Threading + 4 E-Cores)  
**Máquina Virtual (Guest):** `Windows10_Desarrollo` (UEFI OVMF SecureBoot, TPM 2.0, VirtIO)  
**Almacenamiento:** QCOW2 sobre NVMe  

---

## 1. Diagnóstico Inicial y Cuellos de Botella Detectados

Al auditar la máquina virtual en ejecución previa, se identificaron varios problemas que degradaban el rendimiento general tanto del sistema invitado (Windows) como del sistema anfitrión (Linux):

1. **Incompatibilidad con Arquitectura Híbrida (Heterogénea):**
   * El procesador cuenta con dos tipos de núcleos: **P-Cores (Performance)** hasta 4.6 GHz con caché L2 dedicada y **E-Cores (Efficiency)** hasta 3.4 GHz con caché L2 compartida.
   * La VM tenía 8 vCPUs asignados sin fijación (*CPU Pinning*). El planificador del kernel Linux migraba aleatoriamente los hilos entre núcleos de alta y baja velocidad, provocando *thread jitter*, latencia de sincronización e invalidación continua de caché L1/L2.
   * Con 8 hilos asignados sin límite, Windows y las tareas pesadas de QEMU competían por los mismos recursos que la interfaz gráfica de Linux.

2. **Bloqueo del Bucle Principal de I/O en Disco (QEMU Big Lock):**
   * El controlador VirtIO no contaba con hilos dedicados (`iothreads`). Cada operación de lectura o escritura detenía momentáneamente el hilo principal de emulación, causando microcongelamientos en la interfaz y en el cursor.
   * No se estaba utilizando el motor asíncrono moderno del kernel Linux (`io_uring`).

3. **Sobrecarga de Interrupciones y Reloj (VM-Exits):**
   * El reloj virtual dependía de trampas continuas al hipervisor KVM en lugar de confiar en el contador de marcas de tiempo invariante del procesador (*Invariant TSC*).
   * La configuración incluía la bandera `avic` (exclusiva de AMD) en una CPU Intel.

4. **Falta del Canal de Comunicación con el Agente Invitado:**
   * Existía el canal SPICE (`com.redhat.spice.0`), pero no el canal de sockets de `qemu-guest-agent` (`org.qemu.guest_agent.0`), impidiendo operaciones de recorte de disco (`fstrim`) y sincronización limpia de hora.

5. **Límite Gráfico del Adaptador QXL:**
   * El adaptador de video configurado era QXL (diseñado para aceleración 2D antigua). Windows 10/11 delega la composición visual (*Desktop Window Manager - DWM*) a Direct3D. Al carecer de aceleración 3D, Windows renderizaba la interfaz por software (CPU WARP) a 20-30 FPS.

---

## 2. Mapa de Núcleos del Host y Estrategia de Balance

El procesador **Intel Core i5-13420H** organiza sus 12 hilos lógicos de la siguiente forma:

| Hilos Lógicos | Núcleo Físico | Tipo de Núcleo | Frecuencia Máx. | Uso Asignado |
| :--- | :--- | :--- | :--- | :--- |
| **0 y 1** | Núcleo 0 | P-Core (SMT) | 4.6 GHz | **vCPU 0 y vCPU 1 (Windows)** |
| **2 y 3** | Núcleo 1 | P-Core (SMT) | 4.6 GHz | **vCPU 2 y vCPU 3 (Windows)** |
| **4 y 5** | Núcleo 2 | P-Core (SMT) | 4.6 GHz | **vCPU 4 y vCPU 5 (Windows)** |
| **6 y 7** | Núcleo 3 | P-Core (SMT) | 4.6 GHz | **Host Linux (P-Core Reservado)** |
| **8, 9, 10, 11** | Núcleos 4, 5, 6, 7 | E-Cores | 3.4 GHz | **Host Linux + Emulador QEMU + I/O** |

### Distribución 50/50:
* **Invitado Windows (6 vCPUs):** Utiliza **3 P-Cores enteros (hilos 0 a 5)**. Disfruta de la máxima frecuencia monohilo, cachés L2 dedicadas de 1.25 MB y pares Hyper-Threading reales.
* **Anfitrión Linux (6 hilos):** Mantiene **1 P-Core entero (hilos 6 y 7)** para garantizar que el entorno de escritorio (KDE/Wayland), navegadores y aplicaciones pesadas respondan a 4.6 GHz sin latencia, más los **4 E-Cores (hilos 8 al 11)** para servicios del sistema y compilaciones en segundo plano.

---

## 3. Detalle Técnico de los Cambios en el XML de libvirt

### 3.1. Asignación y Afinidad de CPU (`<vcpu>` y `<cputune>`)
```xml
  <vcpu placement='static'>6</vcpu>
  <iothreads>1</iothreads>
  <cputune>
    <vcpupin vcpu='0' cpuset='0'/>
    <vcpupin vcpu='1' cpuset='1'/>
    <vcpupin vcpu='2' cpuset='2'/>
    <vcpupin vcpu='3' cpuset='3'/>
    <vcpupin vcpu='4' cpuset='4'/>
    <vcpupin vcpu='5' cpuset='5'/>
    <emulatorpin cpuset='6-11'/>
    <iothreadpin iothread='1' cpuset='6-11'/>
  </cputune>
```
* `vcpupin`: Amarra cada vCPU a un hilo físico específico de los P-Cores.
* `emulatorpin`: Restringe el hilo de emulación de QEMU a los núcleos 6-11, evitando que le robe ciclos a los vCPUs de Windows.
* `iothreadpin`: Fuerza a que las transferencias de disco corran en los núcleos del host (6-11).

### 3.2. Topología y Reloj Hardware Invariante (`<cpu>` y `<clock>`)
```xml
  <cpu mode='host-passthrough' check='none' migratable='off'>
    <topology sockets='1' dies='1' clusters='1' cores='3' threads='2'/>
    <feature policy='require' name='invtsc'/>
  </cpu>
  <clock offset='localtime'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
    <timer name='hypervclock' present='yes'/>
    <timer name='tsc' present='yes' mode='native'/>
  </clock>
```
* `topology`: Expone a Windows una estructura realista de 1 socket con 3 núcleos y 2 hilos por núcleo, permitiendo que el programador interno de Windows organice sus procesos de forma óptima.
* `invtsc` (Invariant TSC) con `migratable='off'`: Informa al sistema operativo invitado que la frecuencia del temporizador TSC nunca cambia con estados de ahorro de energía. Windows lee el reloj directamente del silicio (`mode='native'`), eliminando miles de interrupciones por segundo.

### 3.3. Depuración de Características Hyper-V (`<hyperv>`)
```xml
    <hyperv mode='custom'>
      <relaxed state='on'/>
      <vapic state='on'/>
      <spinlocks state='on' retries='8191'/>
      <vpindex state='on'/>
      <runtime state='on'/>
      <synic state='on'/>
      <stimer state='on'>
        <direct state='on'/>
      </stimer>
      <frequencies state='on'/>
      <tlbflush state='on'/>
      <ipi state='on'/>
      <evmcs state='on'/>
    </hyperv>
```
* Se retiró `<avic state='on'/>` ya que AVIC (*Advanced Virtual Interrupt Controller*) es exclusivo de procesadores AMD. En Intel, el kernel de Linux maneja las interrupciones mediante APICv por defecto.
* Se agregó `<stimer state='on'><direct state='on'/></stimer>`, lo cual permite que los temporizadores sintéticos de Hyper-V envíen interrupciones directas al procesador sin requerir *hypercalls* intermedias.

### 3.4. Controlador de Disco con `io_uring` y Hilo Dedicado
```xml
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none' io='io_uring' discard='unmap' iothread='1'/>
      <source file='/var/lib/libvirt/images/win11.qcow2'/>
      <target dev='vda' bus='virtio'/>
      <boot order='1'/>
    </disk>
```
* `io='io_uring'`: Aprovecha el subsistema asíncrono de alto rendimiento del kernel Linux para I/O.
* `iothread='1'`: Desacopla la cola de peticiones de VirtIO del bucle principal del emulador, canalizándolas a través del hilo auxiliar dedicado.
* `discard='unmap'`: Permite que Windows informe los bloques liberados para que el archivo `qcow2` no crezca innecesariamente.

### 3.5. Canal para el Agente QEMU
```xml
    <channel type='unix'>
      <target type='virtio' name='org.qemu.guest_agent.0'/>
      <address type='virtio-serial' controller='0' bus='0' port='2'/>
    </channel>
```
* Proporciona el socket Unix de comunicación entre el servicio `qemu-guest-agent` dentro de Windows y el demonio `libvirtd` en Linux.

---

## 4. Solución al Rendimiento Gráfico: RDP vs Visor SPICE

Aunque los controladores VirtIO y SPICE permiten redimensionar pantalla y compartir portapapeles, el adaptador de video virtual **QXL carece de aceleración 3D Direct3D**. 

Para obtener **60 FPS fluidos**, latencia imperceptible en el cursor y renderizado suave en navegadores:

1. **En Windows:**  
   * Ir a *Configuración > Sistema > Escritorio Remoto* y activar la casilla **Habilitar escritorio remoto**.
2. **En Manjaro Linux:**  
   * Abrir **Remmina** (ya preinstalado en el sistema).
   * Crear un nuevo perfil con protocolo **RDP**.
   * Servidor: Introducir la IP local de la VM (por ejemplo `192.168.122.232`).
   * Pestaña *Avanzado*:
     * Calidad: *Mejor calidad (LAN)*.
     * Códec: Seleccionar *H.264 (AVC444)* o *GFX*.
3. **Resultado:**  
   RDP utiliza el pipeline de renderizado optimizado de Windows con decodificación cliente en Linux, eliminando por completo la carga innecesaria de CPU en la emulación de video.

---

## 5. Comandos Útiles de Comprobación y Monitoreo

### Verificación de afinidad de hilos en tiempo real:
```bash
# Comprobar que los vCPUs 0 a 5 corren estrictamente en los núcleos 0 a 5 del host:
ps -T -p $(pgrep -f "guest=Windows10_Desarrollo" | head -n 1) -o pid,tid,psr,comm
```

### Ejecutar TRIM manual desde el anfitrión:
*(Requiere el servicio qemu-guest-agent en Windows activo)*
```bash
virsh -c qemu:///system domfstrim Windows10_Desarrollo
```

### Consultar estado del agente de la máquina:
```bash
virsh -c qemu:///system qemu-agent-command Windows10_Desarrollo '{"execute":"guest-info"}'
```

### Realizar copias de seguridad de la configuración XML:
```bash
virsh -c qemu:///system dumpxml Windows10_Desarrollo > ~/backup_Windows10_Desarrollo.xml
```
