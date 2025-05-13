<div class="header">
  <div class="header-content">
    <h3>FT GP 008_V4 DOCUMENTO DE APOYO</h3>
    <p>30/12/2024</p>
    <p>Acceso: Interno</p>
  </div>
  <div class="header-logo">
    <img src="/logos/ti724-adaptamos-la-tecnologia-a-tu-empresa-colombia.png" alt="Logo TI 724" class="logo">
  </div>
</div>

# Guía Paso a Paso para Implementar la Extensión SAP ABAP en Dynatrace

Esta guía proporciona instrucciones detalladas para implementar la extensión de SAP ABAP en Dynatrace, permitiendo el monitoreo de sistemas SAP NetWeaver ABAP (versión 7.31 o superior). Incluye ejemplos prácticos para configuraciones, comandos y verificaciones. La implementación involucra configuraciones en el sistema SAP, el ActiveGate de Dynatrace y la interfaz de Dynatrace.

---

## **Paso 1: Verificar Requisitos del Sistema SAP**

**Objetivo**: Confirmar que el sistema SAP cumple con los requisitos mínimos.

1. **Verificar la Versión de SAP NetWeaver**:
   - Inicia sesión en el sistema SAP.
   - Ejecuta la transacción `SM51` o ve a **System > Status** para verificar la versión.
   - **Requisito**: SAP NetWeaver ABAP 7.31 o superior.
   - **Ejemplo**:
     - En **System > Status**, observas: `SAP_BASIS 740`. Esto indica compatibilidad (7.40 > 7.31).

2. **Recopilar Información del Sistema**:
   - Anota los siguientes detalles clave, necesarios para configurar la conexión en Dynatrace:
     - **System ID**: Identificador único del sistema SAP (ejemplo: `PRD`).
     - **Instance ID**: Número de instancia del servidor de aplicaciones (un número de dos dígitos, ejemplo: `00`).
     - **SAP Application Server Address**: La dirección IP o nombre de host del servidor de aplicaciones SAP.
     - **Message Server (si aplica)**: La dirección IP o nombre de host del Message Server y su puerto, si el sistema usa un Message Server para conectar instancias.
   - **Cómo obtener el SAP Application Server Address**:
     - **Método 1: Transacción `SM51`**:
       - Ejecuta `SM51` en SAP.
       - En la lista de servidores, identifica el servidor de aplicaciones que deseas monitorear.
       - La columna **Host** muestra el nombre de host (ejemplo: `sap-prod.example.com`) o la dirección IP (ejemplo: `192.168.1.100`).
       - **Ejemplo**: En `SM51`, ves `Host: sap-prod.example.com`, `Instance: PRD_DVEBMGS00`. El **SAP Application Server Address** es `sap-prod.example.com` y el **Instance ID** es `00`.
     - **Método 2: Transacción `RZ11`**:
       - Ejecuta `RZ11` y busca el parámetro `SAPLOCALHOST` o `SAPLOCALHOSTFULL`.
       - El valor indica el nombre de host del servidor (ejemplo: `sap-prod.example.com`).
     - **Método 3: Consulta al Administrador de SAP**:
       - Si no tienes acceso a `SM51` o `RZ11`, solicita al administrador de SAP la dirección IP o nombre de host del servidor de aplicaciones.
       - Proporciónale el **System ID** (ejemplo: `PRD`) para que identifique el servidor correcto.
     - **Verificación**:
       - Desde el servidor del ActiveGate, prueba la conectividad con:
         ```bash
         ping sap-prod.example.com
         # o
         ping 192.168.1.100
         ```
       - Asegúrate de que el nombre de host sea resoluble (si usas DNS) o usa la IP directamente.
   - **Cómo obtener los Detalles del Message Server (si aplica)**:
     - **Cuándo aplica**: Si el sistema SAP usa un Message Server para gestionar múltiples instancias (común en entornos con clústeres), necesitas los detalles del Message Server en lugar del SAP Application Server Address.
     - **Método 1: Transacción `SM51`**:
       - En `SM51`, busca el servidor marcado como **Message Server** en la columna **Type**.
       - Anota el **Host** (ejemplo: `msg-prod.example.com`) y el **System Number** (ejemplo: `01`).
       - El puerto del Message Server suele ser `36<system_number>` (ejemplo: `3601` para el System Number `01`).
       - **Ejemplo**: En `SM51`, ves un servidor con `Type: Message Server`, `Host: msg-prod.example.com`, `System Number: 01`. El **Message Server Host** es `msg-prod.example.com` y el puerto es `3601`.
     - **Método 2: Transacción `SMLG`**:
       - Ejecuta `SMLG` para ver los grupos de logon y el Message Server asociado.
       - Selecciona un grupo (ejemplo: `PUBLIC`) y haz doble clic para ver el **Message Server Host** y el puerto.
       - **Ejemplo**: En `SMLG`, el grupo `PUBLIC` muestra `Message Server: msg-prod.example.com`, `Port: 3601`.
     - **Método 3: Perfil del Sistema**:
       - Usa la transacción `RZ10` para revisar los parámetros del perfil del sistema.
       - Busca parámetros como `rdisp/mshost` (nombre del Message Server) y `rdisp/msserv` (puerto).
       - **Ejemplo**: En `RZ10`, `rdisp/mshost = msg-prod.example.com`, `rdisp/msserv = 3601`.
     - **Método 4: Consulta al Administrador de SAP**:
       - Si no tienes acceso a estas transacciones, pide al administrador el nombre de host o IP del Message Server, el puerto y el grupo de servidores (ejemplo: `PUBLIC`).
     - **Verificación**:
       - Prueba la conectividad desde el ActiveGate:
         ```bash
         telnet msg-prod.example.com 3601
         ```
       - Si el puerto no está abierto, coordina con el equipo de red para habilitarlo.
   - **Ejemplo Final**:
     - **SAP Application Server Address**: `sap-prod.example.com` (obtenido de `SM51`, columna `Host`).
     - **Instance ID**: `00` (obtenido de `SM51`, instancia `PRD_DVEBMGS00`).
     - **Message Server (si aplica)**: `msg-prod.example.com`, puerto `3601` (obtenido de `SMLG` o `RZ10`).
     - **System ID**: `PRD` (obtenido de `SM51` o **System > Status**).

3. **Acción**:
   - Si la versión es inferior a 7.31, coordina una actualización con el administrador de SAP.
   - Guarda los detalles recopilados en un lugar accesible para usarlos en la configuración de Dynatrace.

---

## **Paso 2: Preparar el ActiveGate**

**Objetivo**: Instalar y configurar un ActiveGate con un perfil de rendimiento dedicado.

1. **Instalar el ActiveGate**:
   - Accede a Dynatrace (interfaz web).
   - Ve a **Deploy Dynatrace > Install ActiveGate**.
   - Descarga el instalador para el sistema operativo del servidor:
     - **Linux**: `Dynatrace-ActiveGate-Linux-x86_64.sh`.
     - **Windows**: `Dynatrace-ActiveGate-Windows-x86_64.exe`.
   - Instala:
     - **Linux**:
       ```bash
       sudo sh Dynatrace-ActiveGate-Linux-x86_64.sh
       ```
       Sigue las instrucciones, proporcionando el token de instalación desde Dynatrace.
     - **Windows**: Ejecuta el instalador como administrador.
   - **Ejemplo**:
     - Servidor: Ubuntu 20.04, 4 GB RAM, 2 núcleos CPU.
     - Comando ejecutado: `sudo sh Dynatrace-ActiveGate-Linux-x86_64.sh --set-env-id=<ENV_ID> --set-token=<TOKEN>`.

2. **Configurar el Perfil de Rendimiento Dedicado**:
   - En Dynatrace, ve a **Settings > ActiveGates**.
   - Selecciona el ActiveGate instalado.
   - Habilita el perfil para extensiones (bajo “Extension Execution” o “Monitoring”).
   - **Requisito**: Asegúrate de que el servidor tenga al menos **0.5 núcleos CPU** y **1.5 GB RAM** por configuración de monitoreo.
   - **Ejemplo**:
     - Configuración: 1 servidor SAP monitoreado → 0.5 núcleos, 1.5 GB RAM.
     - Verificación en Linux:
       ```bash
       top # Confirma uso de CPU y RAM
       ```

3. **Habilitar RUM Beacon Forwarder (Opcional)**:
   - Si deseas capturar sesiones de usuario (Real User Monitoring), edita el archivo `custom.properties`:
     - **Linux**: `/var/lib/dynatrace/gateway/config/custom.properties`.
     - **Windows**: `C:\ProgramData\dynatrace\gateway\config\custom.properties`.
   - Añade:
     ```ini
     [beacon_forwarder]
     beacon_forwarder_enabled = true
     ```
   - Reinicia el ActiveGate:
     - **Linux**:
       ```bash
       sudo systemctl restart dynatracegateway
       ```
     - **Windows**: Usa el Administrador de Servicios.
   - **Ejemplo**:
     - Archivo editado: `/var/lib/dynatrace/gateway/config/custom.properties`.
     - Verificación: `sudo systemctl status dynatracegateway` muestra el servicio activo.

4. **Verificar Conectividad**:
   - Asegúrate de que el ActiveGate pueda comunicarse con el servidor SAP en el puerto RFC (ej. 3300 para la instancia 00).
   - Prueba:
     ```bash
     telnet sap-prod.example.com 3300
     ```
   - **Ejemplo**:
     - Resultado: `Connected to sap-prod.example.com` → Conexión exitosa.
     - Si falla, coordina con el equipo de red para abrir el puerto.

---

## **Paso 3: Configurar el SAP Java Connector (JCo)**

**Objetivo**: Instalar y configurar el SAP JCo para la comunicación con SAP.

1. **Descargar el SAP JCo**:
   - Accede al **SAP Support Portal** (requiere credenciales).
   - Descarga la versión de 64 bits del SAP JCo 3.x para tu sistema operativo.
   - **Ejemplo**:
     - Archivos descargados: `sapjco3.jar`, `libsapjco3.so` (Linux) o `sapjco3.dll` (Windows).

2. **Colocar los Archivos JCo**:
   - Crea una carpeta en el servidor del ActiveGate:
     - **Linux**: `/opt/dynatrace/jco/`.
     - **Windows**: `C:\Dynatrace\JCo\`.
   - Copia los archivos:
     - `sapjco3.jar`.
     - `libsapjco3.so` (Linux) o `sapjco3.dll` (Windows).
   - Configura permisos:
     - **Linux**:
       ```bash
       sudo mkdir -p /opt/dynatrace/jco
       sudo cp sapjco3.jar libsapjco3.so /opt/dynatrace/jco/
       sudo chmod -R 755 /opt/dynatrace/jco
       sudo chown dtuser:dtuser /opt/dynatrace/jco
       ```
     - **Windows**: Asegúrate de que el usuario del servicio ActiveGate tenga acceso.
   - **Ejemplo**:
     - Ruta final: `/opt/dynatrace/jco/sapjco3.jar`, `/opt/dynatrace/jco/libsapjco
