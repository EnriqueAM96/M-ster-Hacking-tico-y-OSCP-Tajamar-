
## 1. Preparación del Entorno
  
### Actualizamos el sistema operativo

```bash
sudo apt update && sudo apt upgrade -y
```

### Instalación paquetes requeridos

```bash
sudo apt install apt-transport-https curl gnupg2 -y
```

### Importar la clave GPG de Elastic

```bash
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg
```

### Añadir el repositorio de Elastic

```bash
echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```

### Actualizar la lista de paquetes disponibles

```bash
sudo apt update
```

## 2. Instalación de Elasticsearch

### Instalar Elasticsearch

```bash
sudo apt install elasticsearch -y
```

Durante la instalación, se generará automáticamente una contraseña para el usuario **elastic**

```
--------------------- Security autoconfiguration information ----------------------

Authentication and authorization are enabled.

TLS for the transport and HTTP layers is enabled and configured.

The generated password for the elastic built-in superuser is : CONTRASEÑA
```

### Configurar Elasticsearch

Edita el archivo de configuración principal:

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Ajusta los siguientes parámetros *`Quita  el "#" de las lineas a modificar`*:

```yaml
# ---------------------------------- Cluster -----------------------------------
cluster.name: elk-cluster #--> Identificador del cluster

# ------------------------------------ Node ------------------------------------
node.name: node-1 #--> nombre del nodo actual

# ----------------------------------- Network ----------------------------------
network.host: localhost #--> Define desde dónde se aceptan las coneexiones (0.0.0.0 = todas las redes)

http.port: 9200 #--> Puerto HTTP por defecto de Elasticsearch

# --------------------------------- Discovery ----------------------------------
discovery.type: single-node #--> Modo de nodo único

xpack.security.enabled: true 

xpack.security.enrollment.enabled: true 
```

### Iniciar y habilitar el servicio 

```bash
sudo systemctl daemon-reload

sudo systemctl enable elasticsearch

sudo systemctl start elasticsearch
```

### Comprobar estado

```bash
sudo systemctl status elasticsearch
```


### Probar la conexión (espera 30-60 segundos después de iniciar)

```bash
curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:TU_CONTRASEÑA https://localhost:9200
```
  
Si olvidas la contraseña

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

## 3. Instalación de Kibana

### Instalar Kibana

```bash
sudo apt install kibana -y
```

### Crear token de inscripción para Kibana

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```

**Guardar el token generado para configurar Kibana**.

### Configurar Kibana

Abre el archivo de configuración:

```bash
sudo nano /etc/kibana/kibana.yml
```

Agrega los siguientes parámetros:

```yaml
server.port: 5601

server.host: "0.0.0.0"

server.name: "kibana-server"

elasticsearch.hosts: ["https://localhost:9200"]
```

### Iniciar y habilitar Kibana

```bash
sudo systemctl daemon-reload

sudo systemctl enable kibana

sudo systemctl start kibana
```

### Revisar el estado del servicio 

```bash
sudo systemctl status kibana
```

### Configurar Kibana desde el navegador

1. Abre tu navegador y accede a: `http://IP_DE_TU_SERVIDOR:5601`

2. PIntroduce el token generado anteriormente

3. Obtén el código de verificación ejecutando:

```bash
sudo /usr/share/kibana/bin/kibana-verification-code
```

4. Escribe el código en la interfaz web

5. Inicia sesión con el usuario `elastic` y la contraseña generada durante la instalación de Elasticsearch

## 4. Instalar Elastic Agent con Fleet Server

### Acceder a Fleet desde Kibana

1. En Kibana, ve a **☰ Menu** → **Management** → **Manage** → **Fleet**
2. Si es primera vez, verás el asistente de configuración
3. Click en **Add Fleet Server**

### Generar token de Fleet Server

Copia el comando que te mostrara Kibana

### Descargar Elastic Agent

```bash
cd /tmp curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.15.0-linux-x86_64.tar.gz tar xzvf elastic-agent-8.15.0-linux-x86_64.tar.gz cd elastic-agent-8.15.0-linux-x86_64
```

### Instalar como Fleet Server

Ejecuta el comando copiado de Kibana. Por ejemplo:

```bash
sudo ./elastic-agent install \ --fleet-server-es=https://localhost:9200 \ --fleet-server-service-token=TU_TOKEN_AQUI \ --fleet-server-policy=fleet-server-policy \ --fleet-server-es-ca=/etc/elasticsearch/certs/http_ca.crt \ --fleet-server-port=8220
```

### Verificar la instalación del agente

```bash
sudo elastic-agent status
```

## 5. Añadir la Integración de Elastic Defend

### Crear una nueva política de agente

### Paso 1 instalar un nuevo Elastic Agent

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.15.0-linux-x86_64.tar.gz

tar xzvf elastic-agent-8.15.0-linux-x86_64.tar.gz

cd elastic-agent-8.15.0-linux-x86_64

sudo ./elastic-agent install \

  --url=https://IP_FLEET_SERVER:8220 \

  --enrollment-token=TU_TOKEN_DE_INSCRIPCION
```

### Paso 2 integramos la nueva agent policy para Elastic Defend

1. En Kibana, ve a **Management** → **Fleet** → **Agent policies**

2. Click en **Create agent policy**

3. Nombre: `security-policy` 

4. Descripción: `Política para agentes de Elastic Defend`

5. Click en **Create agent policy**

### Añadir la integración Elastic Defend 

1. Ve a **Management** → **Integrations**

2. Busca **Elastic Defend** (es la integración de seguridad de endpoints)

3. Seleccionar **Add Elastic Defend**

4. Configura:

   - **Integration name**: `nombre_personalizado`

   - **Configuration:** `Traditional Endpoints` → `Complete EDR`

5. Selecciona la política `Fleet Server Policy`

6. Guarda los cambios y continua

## 6. Recoleccion de datos de Seguridad del Host

### Verificar que Elastic Defend está activo

1. En Kibana, ve a **Security** → **Endpoints**

2. El host debe aparecer como "Healthy"

3. Click en el host para ver detalles

### Configurar políticas de detección

1. Ve a **Security** → **Manage** → **Policies**

2. Elige `security-policy`

3. Ajusta las opciones:

#### Proteccion de Malware

- **Mode**: Prevent 

- **Notification**: On

#### Proteccion contra el ransomware

- **Mode**: Prevent

- **Notification**: On

#### Proteccion de memoria

- **Mode**: Prevent

#### Behavior Protection

- **Mode**: Prevent

### Incorporar integraciones adicionales para seguridad

#### System Logs

1. Ve a **Management** → **Integrations**

2. Busca **System**

3. Click en **Add System**

4. Selecciona la política `security-policy`

5. Habilita:

   - **Collect logs from System instances**

   - Syslog

   - Auth logs

   - System logs

6. Activa **Collect metrics** para recopilar métricas del sistema (opcional)

#### Auditd (Auditoría en Linux)

1. Instala auditd en el sistema:

```bash
sudo apt install auditd -y

sudo systemctl enable auditd

sudo systemctl start auditd
```

2. En Kibana, añade la integración **Auditd**:

   - Ve a **Management** → **Integrations**

   - Busca **Auditd**

   - Click en **Add Auditd**

   - Selecciona `security-policy`

   - Indica las rutas de los archivos que se desean auditar

#### Crear reglas personalizadas de auditoría

Edita `/etc/audit/rules.d/audit.rules`:

```bash
sudo nano /etc/audit/rules.d/audit.rules
```

Agrega las reglas de ejemplo:

```bash
# Vigilar cambios en archivos críticos del sistema 
-w /etc/passwd -p wa -k passwd_changes

-w /etc/shadow -p wa -k shadow_changes

-w /etc/sudoers -p wa -k sudoers_changes

# Registrar comandos ejecutados con privilegios de root 

-a always,exit -F arch=b64 -S execve -F euid=0 -k root_commands

# Detectar modificaciones en configuraciones de red y SSH 

-w /etc/network/ -p wa -k network_changes

-w /etc/ssh/sshd_config -p wa -k sshd_config_changes

# Registrar intentos de inicio de sesión

-w /var/log/lastlog -p wa -k logins

-w /var/run/faillock/ -p wa -k logins

# Monitorear cambios en parámetros del kernel

-w /etc/sysctl.conf -p wa -k sysctl

-a always,exit -F arch=b64 -S init_module -S delete_module -k modules
```

Guarda cambios y reinicia el servicio:

```bash
sudo systemctl restart auditd
```

## 7. Visualizar Datos de Seguridad en Kibana

### Panel de seguridad

1. Ve a **Security** → **Dashboards** → **Overview**

2. En esta vista se podra observar:

   - Eventos de seguridad en tiempo real

   - Alertas generadas

   - Actividad de red

   - Comportamiento de usuarios y hosts

### Exploración de eventos

1. Ve a **Security** → **Events**

2. Utiliza los filtros por:

   - Tipo de evento

   - Host

   - Usuario

   - Proceso

   - Archivo

### Creación de reglas de detección personalizadas

1. Ve a **Security** → **Rules**

2. Click en **Create new rule**

3. Elige el tipo de regla que deseas configurar:

   - Custom query

   - Machine Learning

   - Threshold

   - Event Correlation

4. Define las condiciones de activación

5. Configura las acciones a ejecutar (alertas, notificaciones)

## 8. Comandos Útiles de Mantenimiento

### Comprobar el estado de los servicios

```bash
sudo systemctl status elasticsearch

sudo systemctl status kibana

sudo elastic-agent status
```

### Consultar logs de Elasticsearch

```bash
sudo journalctl -u elasticsearch -f

sudo tail -f /var/log/elasticsearch/elk-cluster.log
```

### Consultar logs de Kibana

```bash
sudo journalctl -u kibana -f

sudo tail -f /var/log/kibana/kibana.log
```


### Consultar logs de Elastic Agent

```bash
sudo tail -f /opt/Elastic/Agent/data/elastic-agent-*/logs/elastic-agent-json.log
```  

### Reiniciar servicios principales

```bash
sudo systemctl restart elasticsearch

sudo systemctl restart kibana

sudo systemctl restart elastic-agent
``` 

### Verificar el estado del agente

```bash
sudo elastic-agent status
``` 

### Restablecer contraseña del usuario elastic

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

## 9. Exposición Segura de Kibana (OPCIONAL - Para acceso remoto)

### Acceso desde Red Local (Sin HTTPS)

```bash
sudo nano /etc/kibana/kibana.yml
```

Configura los siguientes parámetros:

```yaml
server.host: "0.0.0.0"  # Permite acceso desde cualquier IP

server.port: 5601
```

Reinicia el servicio:

```bash
sudo systemctl restart kibana
```

**Configura el firewall:**

```bash
# Si usas UFW
sudo ufw allow from 192.168.1.0/24 to any port 5601  # Ajusta tu red

sudo ufw reload

# O si usas iptables
sudo iptables -A INPUT -p tcp -s 192.168.1.0/24 --dport 5601 -j ACCEPT
```

Ahora puedes acceder desde cualquier equipo de tu red local:

```
http://IP_DEL_SERVIDOR:5601
```

### Comprobación de Seguridad

#### 1. Verificar puertos abiertos

```bash
sudo ss -tulpn | grep -E '5601|443|80'
```

#### 2. Conexión SSL

```bash
# Verificar certificado

openssl s_client -connect localhost:5601 -showcerts 

# O con curl

curl -k https://localhost:5601
```

#### 3. Probar acceso remoto desde otro equipo

```bash
# Desde otro equipo en la red

curl -k https://IP_DEL_SERVIDOR:5601/api/status
```