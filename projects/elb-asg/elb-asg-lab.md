# Lab: Application Load Balancer + Auto Scaling Group en AWS

## 1. Objetivo

El objetivo de este laboratorio es desplegar una aplicación web simple en instancias EC2 gestionadas por un Auto Scaling Group (ASG) y expuestas mediante un Application Load Balancer (ALB).

El lab demuestra los siguientes conceptos:

- Distribución de tráfico con un Application Load Balancer.
- Registro de instancias EC2 en un Target Group.
- Creación automática de instancias mediante un Auto Scaling Group.
- Uso de Launch Template con User Data.
- Health checks del Target Group.
- Relación entre Security Groups, conectividad outbound e instalación de dependencias.
- Troubleshooting de targets en estado `unhealthy`.

---

## 2. Arquitectura

```text
Internet
   |
   v
Application Load Balancer
   |
   v
Target Group
   |
   v
Auto Scaling Group
   |
   v
EC2 Instances en varias Availability Zones
```

Arquitectura usada en este lab:

- ALB público en subnets públicas.
- EC2 lanzadas por un Auto Scaling Group.
- EC2 en subnets públicas.
- EC2 con public IPv4 habilitada desde el Launch Template.
- Servidor web Apache instalado mediante User Data.
- Target Group con health checks HTTP en el puerto 80.

---

## 3. Servicios utilizados

- Amazon EC2
- Launch Template
- Auto Scaling Group
- Application Load Balancer
- Target Group
- Security Groups
- CloudWatch / métricas del Target Group
- User Data
- Amazon Linux 2023
- Apache HTTP Server (`httpd`)

---

## 4. Flujo general del laboratorio

1. Crear un Launch Template para las instancias EC2.
2. Configurar User Data para instalar y arrancar Apache.
3. Crear un Target Group HTTP en el puerto 80.
4. Crear un Application Load Balancer público.
5. Crear un Auto Scaling Group usando el Launch Template.
6. Asociar el ASG al Target Group.
7. Comprobar que las instancias pasan a estado `healthy`.
8. Probar el acceso a la aplicación mediante el DNS del ALB.
9. Resolver errores de conectividad y health checks.

---

## 5. Launch Template

Se creó un Launch Template con una AMI de Amazon Linux 2023 y un tipo de instancia pequeño para pruebas.

Configuración relevante:

- AMI: Amazon Linux 2023
- Instance type: `t3.micro`
- Security Group: Security Group específico para EC2
- Public IPv4: habilitada
- User Data: instalación de Apache y creación de una página HTML simple

### User Data utilizado

```bash
#!/bin/bash
dnf update -y
dnf install -y httpd
systemctl enable httpd
systemctl start httpd

TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)

AZ=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)

cat <<HTML > /var/www/html/index.html
<h1>ELB + ASG Lab</h1>
<p>Instance ID: $INSTANCE_ID</p>
<p>Availability Zone: $AZ</p>
HTML
```

Este User Data permite identificar qué instancia está respondiendo detrás del Load Balancer.

---

## 6. Security Groups

### Security Group del ALB

Inbound:

```text
HTTP 80 desde 0.0.0.0/0
```

Outbound:

```text
HTTP 80 hacia el Security Group de las EC2
```

También puede dejarse temporalmente como:

```text
All traffic hacia 0.0.0.0/0
```

para simplificar el laboratorio.

### Security Group de las EC2

Inbound:

```text
HTTP 80 desde el Security Group del ALB
SSH 22 desde mi IP pública /32, solo para troubleshooting
```

Outbound:

```text
All traffic hacia 0.0.0.0/0
```

Esta regla outbound fue necesaria para que las instancias pudieran descargar paquetes durante la ejecución del User Data.

---

## 7. Target Group

Configuración del Target Group:

- Target type: Instances
- Protocol: HTTP
- Port: 80
- Health check protocol: HTTP
- Health check path: `/`
- Health check port: Traffic port

El Auto Scaling Group registra automáticamente las instancias en el Target Group.

---

## 8. Application Load Balancer

Configuración del ALB:

- Type: Application Load Balancer
- Scheme: Internet-facing
- Listener: HTTP 80
- Forwarding: hacia el Target Group
- Subnets: públicas, en al menos dos Availability Zones

El ALB recibe tráfico desde Internet y lo reenvía a las instancias EC2 usando sus IPs privadas dentro de la VPC.

Punto importante:

> Las instancias EC2 no necesitan IP pública para recibir tráfico desde el ALB. El ALB se comunica con ellas usando conectividad privada dentro de la VPC.

---

## 9. Auto Scaling Group

Configuración del ASG:

- Basado en el Launch Template
- Desired capacity: 2
- Minimum capacity: 2
- Maximum capacity: 4
- Asociado al Target Group
- Health checks: ELB
- Subnets: subnets públicas usadas en el lab

El ASG se encarga de mantener el número deseado de instancias y registrar las nuevas instancias en el Target Group.

---

## 10. Pruebas realizadas

### 10.1 Comprobación de instancias creadas por el ASG

Se verificó que el ASG lanzaba correctamente las instancias EC2 según la capacidad deseada.

Resultado esperado:

```text
Desired capacity: 2
Running instances: 2
```

### 10.2 Comprobación del Target Group

Se revisó el estado de los targets en el Target Group.

Resultado esperado tras corregir los problemas:

```text
initial -> healthy
```

### 10.3 Comprobación del ALB

Se accedió al DNS del Application Load Balancer desde el navegador.

Resultado esperado:

```text
ELB + ASG Lab
Instance ID: i-xxxxxxxxxxxxxxxxx
Availability Zone: xx-xxxx-x
```

Al refrescar varias veces, se puede comprobar que el ALB reparte tráfico entre diferentes instancias.

También se puede probar con `curl`:

```bash
for i in {1..10}; do curl http://DNS-DEL-ALB; done
```

---

## 11. Problemas encontrados

### Problema 1: targets en estado `unhealthy`

Inicialmente, las dos instancias del Auto Scaling Group aparecían como `unhealthy` en el Target Group.

Síntoma:

```text
Target Group -> Targets -> unhealthy
```

Posibles causas revisadas:

- Security Group de EC2 no permitía tráfico desde el ALB.
- Security Group del ALB no podía enviar tráfico a las EC2.
- Apache no estaba instalado o no estaba levantado.
- Health check path incorrecto.
- Instancias sin conectividad outbound para ejecutar el User Data.
- Instancias sin public IPv4 en una subnet pública sin NAT Gateway.

Causa real:

```text
Las instancias no podían salir a Internet.
```

Esto ocurrió por dos motivos:

1. El Security Group de las EC2 no tenía outbound hacia Internet.
2. El Launch Template no tenía habilitada la asignación de public IPv4.

Como consecuencia, el User Data no pudo ejecutar correctamente:

```bash
dnf update -y
dnf install -y httpd
```

Por tanto, Apache no se instaló, el puerto 80 no respondía y el Target Group marcaba las instancias como `unhealthy`.

---

## 12. Solución aplicada

### 12.1 Corregir outbound en el Security Group de EC2

Se añadió la siguiente regla outbound al Security Group de las instancias EC2:

```text
All traffic -> 0.0.0.0/0
```

Esto permitió que las instancias pudieran salir a Internet para descargar paquetes.

### 12.2 Corregir el Launch Template

Se modificó el Launch Template para habilitar public IPv4 en las instancias lanzadas por el ASG.

Configuración corregida:

```text
Auto-assign public IP: Enable
```

Después se creó una nueva versión del Launch Template y se actualizó el Auto Scaling Group para usar esa versión.

### 12.3 Relanzar instancias

Como el User Data normalmente se ejecuta solo durante el primer arranque, las instancias antiguas no reinstalan Apache automáticamente al corregir el Security Group.

Por eso se terminaron las instancias existentes y se dejó que el Auto Scaling Group lanzara nuevas instancias.

Resultado:

```text
ASG lanza nuevas EC2
User Data se ejecuta correctamente
httpd se instala y arranca
Target Group pasa de initial a healthy
ALB responde correctamente
```

---

## 13. Explicación del fallo

El fallo no estaba en el ALB comunicándose con las EC2. El ALB puede comunicarse con las instancias usando sus IPs privadas dentro de la VPC.

El problema estaba en la conectividad saliente de las EC2.

Flujo fallido inicial:

```text
ASG lanza EC2
   |
   v
EC2 ejecuta User Data
   |
   v
User Data intenta instalar httpd
   |
   v
EC2 no tiene salida a Internet
   |
   v
Apache no se instala
   |
   v
Puerto 80 no responde
   |
   v
Target Group marca unhealthy
```

Flujo corregido:

```text
ASG lanza EC2 con public IPv4
   |
   v
Security Group permite outbound
   |
   v
User Data instala httpd
   |
   v
Apache escucha en puerto 80
   |
   v
Target Group health check responde OK
   |
   v
Target pasa a healthy
```

---

## 14. Conceptos importantes aprendidos

### Inbound vs Outbound

Inbound:

```text
Tráfico que entra a un recurso.
```

Outbound:

```text
Tráfico que sale de un recurso.
```

Ejemplo en este lab:

```text
Internet -> ALB -> EC2
```

- El ALB necesita inbound HTTP desde Internet.
- La EC2 necesita inbound HTTP desde el Security Group del ALB.
- La EC2 necesita outbound para descargar paquetes durante el User Data.

### ALB hacia EC2

El ALB no necesita que las EC2 tengan IP pública para enviarles tráfico.

```text
ALB -> EC2 por IP privada
```

### EC2 hacia Internet

Si una EC2 está en una subnet pública pero no tiene public IPv4, no puede salir directamente a Internet mediante un Internet Gateway.

Para que una EC2 sin IP pública salga a Internet, haría falta una arquitectura con NAT Gateway:

```text
Private EC2 -> NAT Gateway -> Internet Gateway -> Internet
```

En este lab se usó la opción más simple:

```text
EC2 en subnet pública + public IPv4 + outbound permitido
```

### User Data

El User Data normalmente se ejecuta durante el primer arranque de la instancia.

Si falla por falta de conectividad, corregir el Security Group después no reejecuta automáticamente el User Data.

Solución práctica en un ASG:

```text
Terminar las instancias antiguas y dejar que el ASG cree nuevas.
```

### Health Checks

El Target Group marca una instancia como `healthy` solo si la aplicación responde correctamente al health check configurado.

Una instancia puede estar `running` en EC2 y al mismo tiempo `unhealthy` en el Target Group.

Esto ocurre porque:

```text
EC2 running = la máquina está encendida
Target healthy = la aplicación responde correctamente
```

---

## 15. Comandos útiles de troubleshooting

Comprobar Apache:

```bash
sudo systemctl status httpd
```

Arrancar Apache manualmente:

```bash
sudo systemctl start httpd
```

Habilitar Apache en el arranque:

```bash
sudo systemctl enable httpd
```

Probar respuesta local:

```bash
curl localhost
```

Instalar Apache manualmente:

```bash
sudo dnf update -y
sudo dnf install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd
```

Probar el ALB varias veces:

```bash
for i in {1..10}; do curl http://DNS-DEL-ALB; done
```

---

## 16. Checklist final

- [x] Launch Template creado.
- [x] User Data configurado.
- [x] Auto-assign public IPv4 habilitado en el Launch Template.
- [x] Security Group del ALB permite HTTP desde Internet.
- [x] Security Group de EC2 permite HTTP desde el Security Group del ALB.
- [x] Security Group de EC2 permite outbound hacia Internet.
- [x] Target Group creado con health check HTTP `/`.
- [x] ASG asociado al Target Group.
- [x] Instancias relanzadas después de corregir la configuración.
- [x] Targets en estado `healthy`.
- [x] ALB responde correctamente desde el navegador.

---

## 17. Lecciones clave para AWS DVA-C02 / SAA-C03

- Un ALB puede enviar tráfico a instancias usando IP privada dentro de la VPC.
- Las EC2 no necesitan IP pública para recibir tráfico desde un ALB.
- Si una EC2 necesita descargar paquetes desde Internet en una subnet pública, necesita public IPv4 y ruta a un Internet Gateway.
- Si una EC2 privada necesita salir a Internet, se debe usar NAT Gateway.
- El Security Group de EC2 debería permitir HTTP desde el Security Group del ALB, no desde todo Internet.
- Los Security Groups son stateful.
- Los cambios en Security Groups se aplican a las instancias existentes.
- El User Data no se reejecuta automáticamente solo porque cambies el Security Group.
- EC2 `running` no significa que la aplicación esté sana.
- Target Group `healthy` significa que la aplicación responde al health check.
- En un ASG, si una instancia falla o se termina, el grupo lanza otra para mantener la capacidad deseada.

---

## 18. Resumen final

En este laboratorio se desplegó una aplicación web simple detrás de un Application Load Balancer y gestionada por un Auto Scaling Group.

El principal problema encontrado fue que las instancias aparecían como `unhealthy` en el Target Group porque Apache no se había instalado correctamente. La causa fue que las EC2 no tenían salida a Internet: faltaba outbound en el Security Group y no estaba habilitada la public IPv4 en el Launch Template.

Tras corregir ambas configuraciones y relanzar las instancias desde el Auto Scaling Group, el User Data se ejecutó correctamente, Apache quedó funcionando y los targets pasaron a estado `healthy`.

Este lab refuerza la diferencia entre conectividad entrante hacia la aplicación y conectividad saliente necesaria para inicializar las instancias.
