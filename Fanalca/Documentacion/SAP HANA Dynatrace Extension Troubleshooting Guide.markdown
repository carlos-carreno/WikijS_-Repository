# Lista de Verificación para Configurar y Diagnosticar la Extensión de Dynatrace para SAP HANA

Este documento detalla los pasos para diagnosticar el error `Cannot connect to jdbc:sap://10.50.0.12:30015/` al configurar la extensión de Dynatrace para monitorear una base de datos SAP HANA, y cómo configurar correctamente la IP de la base de datos. También incluye un script de PowerShell para verificar la configuración en una sola acción.

## Lista de Verificación

### 1. Verificar la Conectividad de Red
**Objetivo**: Confirmar que el ActiveGate puede comunicarse con el servidor SAP HANA en `10.50.0.12:30015`.

- [ ] **Confirmar accesibilidad del host**:
  - Ejecuta en PowerShell:
    ```powershell
    Test-Connection -ComputerName 10.50.0.12 -Count 4
    ```
  - **Qué buscar**: Respuesta con tiempos de ping. Si falla, verifica con el equipo de red (firewall, VPN, enrutamiento).
- [ ] **Confirmar puerto `30015` abierto**:
  - Ejecuta:
    ```powershell
    Test-NetConnection -ComputerName 10.50.0.12 -Port 30015
    ```
  - **Qué buscar**: `TcpTestSucceeded: True`. Si es `False`, revisa el firewall en el servidor SAP HANA o la configuración de red.
- [ ] **Validar puerto en SAP HANA**:
  - En SAP HANA, ejecuta:
    ```sql
    SELECT SERVICE_NAME, PORT FROM M_SERVICES WHERE DATABASE = 'YOUR_DATABASE_NAME';
    ```
  - **Qué buscar**: Confirma que el puerto es `30015`. Si es diferente, actualiza la configuración de la extensión.
- **Acción si falla**:
  - Abre el puerto `30015` en el firewall.
  - Corrige restricciones de red.

### 2. Validar Configuración de SAP HANA
**Objetivo**: Asegurar que la base de datos está activa y configurada para conexiones remotas.

- [ ] **Confirmar estado de la base de datos**:
  - En el servidor SAP HANA:
    ```bash
    sapcontrol -nr <instance_number> -function GetSystemInstanceList
    ```
  - **Qué buscar**: Estado `GREEN`. Si no, inicia la base de datos.
- [ ] **Verificar credenciales**:
  - Ejecuta en SAP HANA:
    ```sql
    SELECT * FROM GRANTED_ROLES WHERE GRANTEE = 'YOUR_USERNAME';
    ```
  - Otorga roles si faltan:
    ```sql
    GRANT PUBLIC, MONITORING TO YOUR_USERNAME;
    ```
- [ ] **Probar conexión JDBC**:
  - Usa DBeaver o un script con:
    ```plaintext
    jdbc:sap://10.50.0.12:30015/?databaseName=YOUR_DATABASE_NAME
    ```
  - **Qué buscar**: Conexión exitosa. Si falla, revisa credenciales o nombre de la base.
- **Acción si falla**:
  - Corrige credenciales o nombre de la base.
  - Verifica el parámetro `listeninterface` en `global.ini` de SAP HANA.

### 3. Verificar Controlador JDBC
**Objetivo**: Asegurar que el controlador JDBC está instalado y es compatible.

- [ ] **Confirmar archivo `ngdbc-2.14.7.jar`**:
  - Verifica en:
    - Windows: `C:\ProgramData\dynatrace\remotepluginmodule\agent\conf\userdata\libs`
    - Linux: `/var/lib/dynatrace\remotepluginmodule\agent\conf\userdata\libs`
  - En PowerShell:
    ```powershell
    Test-Path -Path "C:\ProgramData\dynatrace\remotepluginmodule\agent\conf\userdata\libs\ngdbc-2.14.7.jar"
    ```
  - **Qué buscar**: `True`. Si es `False`, copia el archivo.
- [ ] **Validar permisos**:
  - Ejecuta:
    ```powershell
    Get-Acl -Path "C:\ProgramData\dynatrace\remotepluginmodule\agent\conf\userdata\libs\ngdbc-2.14.7.jar" | Format-List
    ```
  - **Qué buscar**: Permisos de lectura para el usuario del ActiveGate.
- [ ] **Verificar compatibilidad**:
  - Confirma que `ngdbc-2.14.7.jar` es compatible con tu versión de SAP HANA.
- **Acción si falla**:
  - Copia el archivo al directorio correcto.
  - Ajusta permisos:
    ```powershell
    icacls "C:\ProgramData\dynatrace\remotepluginmodule\agent\conf\userdata\libs\ngdbc-2.14.7.jar" /grant "Everyone:R"
    ```

### 4. Configurar la IP de la Base de Datos
**Objetivo**: Configurar la extensión en Dynatrace con la IP correcta.

- [ ] **Acceder al Dynatrace Hub**:
  - Ve a **Hub > SAP HANA Database Monitoring**.
- [ ] **Configurar dispositivo**:
  - Completa:
    - **Endpoint**: `10.50.0.12:30015`
    - **Database Name**: Nombre de la base (ej. `HDB`).
    - **Username**: Usuario con roles `PUBLIC` y `MONITORING`.
    - **Password**: Contraseña.
    - **JDBC URL (opcional)**:
      ```plaintext
      jdbc:sap://10.50.0.12:30015/?databaseName=YOUR_DATABASE_NAME
      ```
    - **Table Information**: Desmarca para reducir costos.
  - Asigna el grupo de ActiveGates con el controlador JDBC.
- [ ] **Validar configuración**:
  - Guarda y espera la validación de Dynatrace.
- **Acción si falla**:
  - Revisa errores en Dynatrace.
  - Corrige IP, puerto, nombre de la base o credenciales.

### 5. Verificar ActiveGate
**Objetivo**: Asegurar que el ActiveGate está configurado y en ejecución.

- [ ] **Confirmar extensión activada**:
  - En **Settings > Monitoring > Monitored technologies**, verifica que la extensión esté habilitada.
- [ ] **Reiniciar ActiveGate**:
  - Ejecuta:
    ```powershell
    Restart-Service -Name "Dynatrace Gateway"
    ```
- [ ] **Revisar logs**:
  - Verifica:
    ```powershell
    Get-Content -Path "C:\ProgramData\dynatrace\remotepluginmodule\log\*.log" -Tail 100
    ```
  - **Qué buscar**: Errores JDBC o de conexión.
- **Acción si falla**:
  - Corrige errores en los logs.
  - Asegura acceso a Internet para el ActiveGate.

### 6. Validar Monitoreo
**Objetivo**: Confirmar que las métricas se recopilan.

- [ ] **Revisar métricas**:
  - En **Hosts** o **Databases**, busca `10.50.0.12` o la base SAP HANA.
  - Verifica métricas como CPU, memoria, disco.
- [ ] **Configurar dashboard**:
  - Crea un dashboard para métricas clave.
  - Configura alertas con Davis AI.
- **Acción si falla**:
  - Revisa configuración y logs.
  - Contacta al soporte de Dynatrace.

### 7. Optimizar Costos
**Objetivo**: Reducir consumo de métricas.

- [ ] **Deshabilitar Table Information**:
  - Desmarca en la configuración de la extensión.
- [ ] **Revisar consumo**:
  - Usa la fórmula:
    ```
    (40 + (2 * <# de esquemas>) + (14 * <# de servicios>) + (3 * <# de esquemas> * <promedio de tablas por esquema>))
    ```
- **Acción**:
  - Limita métricas si el consumo es alto.

## Script de PowerShell para Verificaciones Múltiples

Este script realiza las siguientes verificaciones en una sola acción:
- Accesibilidad del host `10.50.0.12`.
- Conexión al puerto `30015`.
- Presencia del controlador JDBC.
- Permisos del controlador.
- Estado del servicio del ActiveGate.
- Errores recientes en los logs del ActiveGate.

Guarda el script como `Check-SAPHANA-Extension.ps1` y ejecútalo en el servidor del ActiveGate con permisos de administrador.

```powershell
# Script para verificar configuración de la extensión de Dynatrace para SAP HANA
$hostIP = "10.50.0.12"
$port = 30015
$jdbcPath = "C:\ProgramData\dynatrace\remotepluginmodule\agent\conf\userdata\libs\ngdbc-2.14.7.jar"
$logPath = "C:\ProgramData\dynatrace\remotepluginmodule\log\*.log"
$serviceName = "Dynatrace Gateway"

Write-Host "=== Verificación de Configuración para SAP HANA ==="

# 1. Verificar accesibilidad del host
Write-Host "`n1. Probando conexión al host $hostIP..."
$pingResult = Test-Connection -ComputerName $hostIP -Count 4 -ErrorAction SilentlyContinue
if ($pingResult) {
    Write-Host "   OK: Host $hostIP es accesible." -ForegroundColor Green
} else {
    Write-Host "   ERROR: Host $hostIP no responde. Verifica red o firewall." -ForegroundColor Red
}

# 2. Verificar puerto
Write-Host "`n2. Probando conexión al puerto $port..."
$portResult = Test-NetConnection -ComputerName $hostIP -Port $port
if ($portResult.TcpTestSucceeded) {
    Write-Host "   OK: Puerto $port está abierto." -ForegroundColor Green
} else {
    Write-Host "   ERROR: Puerto $port no está accesible. Verifica firewall o configuración de SAP HANA." -ForegroundColor Red
}

# 3. Verificar presencia del controlador JDBC
Write-Host "`n3. Verificando controlador JDBC en $jdbcPath..."
if (Test-Path -Path $jdbcPath) {
    Write-Host "   OK: Controlador JDBC encontrado." -ForegroundColor Green
} else {
    Write-Host "   ERROR: Controlador JDBC no encontrado. Copia ngdbc-2.14.7.jar al directorio." -ForegroundColor Red
}

# 4. Verificar permisos del controlador
Write-Host "`n4. Verificando permisos del controlador..."
if (Test-Path -Path $jdbcPath) {
    $acl = Get-Acl -Path $jdbcPath
    $hasReadPermission = $acl.Access | Where-Object { $_.FileSystemRights -match "Read" -and $_.IdentityReference -match "Everyone|SYSTEM" }
    if ($hasReadPermission) {
        Write-Host "   OK: El controlador tiene permisos de lectura adecuados." -ForegroundColor Green
    } else {
        Write-Host "   ERROR: El controlador no tiene permisos de lectura. Ajusta permisos." -ForegroundColor Red
    }
} else {
    Write-Host "   ERROR: No se puede verificar permisos porque el archivo no existe." -ForegroundColor Red
}

# 5. Verificar estado del servicio del ActiveGate
Write-Host "`n5. Verificando estado del servicio $serviceName..."
$service = Get-Service -Name $serviceName -ErrorAction SilentlyContinue
if ($service -and $service.Status -eq "Running") {
    Write-Host "   OK: Servicio $serviceName está en ejecución." -ForegroundColor Green
} else {
    Write-Host "   ERROR: Servicio $serviceName no está en ejecución o no existe. Inicia el servicio." -ForegroundColor Red
}

# 6. Revisar logs del ActiveGate
Write-Host "`n6. Revisando logs recientes del ActiveGate..."
if (Test-Path -Path $logPath) {
    $logContent = Get-Content -Path $logPath -Tail 100 | Select-String "ERROR|JDBC|connection"
    if ($logContent) {
        Write-Host "   ADVERTENCIA: Se encontraron errores en los logs. Detalles:" -ForegroundColor Yellow
        $logContent | ForEach-Object { Write-Host "   $_" }
    } else {
        Write-Host "   OK: No se encontraron errores recientes en los logs." -ForegroundColor Green
    }
} else {
    Write-Host "   ERROR: No se encontraron logs en $logPath." -ForegroundColor Red
}

Write-Host "`n=== Verificación Completada ==="
Write-Host "Sigue la lista de verificación para corregir cualquier error identificado."
```

**Instrucciones**:
1. Guarda el script como `Check-SAPHANA-Extension.ps1`.
2. Ejecútalo en PowerShell con permisos de administrador:
   ```powershell
   .\Check-SAPHANA-Extension.ps1
   ```
3. Revisa la salida para identificar problemas y sigue las acciones recomendadas.