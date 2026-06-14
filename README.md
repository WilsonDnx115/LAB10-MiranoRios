# ENTREGABLES DE RESPUESTAS E INSTRUCCIONES

## PREGUNTAS

## ¿Por qué necesitamos Loki además de Prometheus si ya tenemos /metrics?

Necesitamos Loki porque Prometheus solo maneja métricas cuantitativas, o sea solo números y contadores que te indican con precisión cuándo y cuánto se está saturando o fallando el sistema, pero carece por completo del contexto textual para explicar la causa raíz. Loki, en cambio, actúa como un indexador inteligente de logs que almacena los mensajes de error detallados y los stack traces de la aplicación, permitiéndote investigar el por qué exacto de una falla una vez que Prometheus te ha alertado sobre la anomalía. Ambos se complementan porque las métricas te dan la alarma visual del comportamiento del stack y los logs te entregan el diagnóstico técnico definitivo para solucionar el problema.

## ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código y no creadas a mano?

La ventaja es que conviertes la observabilidad de tu infraestructura en un proceso automatizado, repetible y libre de errores humanos. Al definir las conexiones a Prometheus y Loki mediante archivos de configuración en tu stack de Docker Compose, eliminas la necesidad de ingresar manualmente a la interfaz web de Grafana para rellenar URLs o credenciales cada vez que despliegas el entorno. Esto logra que si necesitas levantar el laboratorio desde cero, clonarlo en otra máquina o migrar entre entornos como DEV y PROD, el sistema de monitoreo estará completamente operativo y conectado desde el primer segundo, manteniendo además un historial de cambios auditable en tu repositorio de Git.

## El panel "CPU contenedor" y el panel "CPU host" pueden mostrar valores muy distintos. ¿Por qué? ¿Cuál usarías para alertar sobre una aplicación concreta?

Muestran valores distintos porque miden el alcance de recursos en niveles de aislamiento completamente diferentes de la infraestructura. El CPU del host calcula el consumo total de la máquina física o virtual completa, sumando el esfuerzo de todos los contenedores activos, procesos del sistema operativo y agentes de recolección, mientras que el CPU del contenedor mide de forma aislada y quirúrgica el uso de recursos que cAdvisor extrae específicamente para un solo proceso, siendo en este laboratorio el contenedor de backend. Para alertar sobre una aplicación usaría el CPU del contenedor, debido a que mide exactamente lo que consume ese servicio, además de que usar la métrica del host generaría falsos positivos si un contenedor vecino o un proceso externo satura la máquina sin que tu servicio sea el causante de la degradación.

## ¿Qué diferencia hay entre el evaluation interval y el pending period de una alarma?

La diferencia radica en que el evaluation interval determina la frecuencia exacta con la que Grafana ejecuta la consulta PromQL o LogQL para revisar si se cumple la condición de la alerta como verificar el estado del consumo cada 30 segundos, mientras que el pending period es el tiempo mínimo que la condición debe mantenerse activa antes de que la alerta pase al estado de disparo definitivo. 

## INSTRUCCIONES

## 1. Clonar el repositorio

Clonar el repositorio y ejecutarlo desde la carpeta principal mediante una terminal basada en WSL con la imagen de Ubuntu, debido a que algunos comandos de Docker no funcionan con el kernel de Windows.

<img width="300" height="387" alt="Image" src="https://github.com/user-attachments/assets/6ea4b39f-e50a-40f0-a4af-e36894e1cd16" />

<img width="745" height="87" alt="Image" src="https://github.com/user-attachments/assets/ebfd0d02-fc49-49f8-bfea-0390b44d7326" />

## 2. Levantar los contenedores

Para levantar los contenedores, colocar el siguiente comando:

```
docker compose up -d --build
```

Al intentar levantarlo, aparecera el siguiente error:

```
Error response from daemon: path / is mounted on / but it is not a shared mount
```
<img width="756" height="106" alt="Captura de pantalla 2026-06-13 212607" src="https://github.com/user-attachments/assets/a9f4f494-cc9d-48ff-9a8e-cc30ae042b5f" />

Esto sucede porque Docker en WSL monta la raíz (/) de forma privada y no compartida, lo que impide que contenedores como node-exporter propaguen sus volúmenes hacia el host, bloqueando su creación con el error. . Para solucionar el problema, en `docker-compose.yml` en el apartado de `volumes` borramos rslav.

```
volumes:
    - /:/host:ro,rslav
```

```
volumes:
    - /:/host:ro
```


Tras esto, colocamos los siguientes comandos en orden respectivamente:

```
curl -fsSL https://get.docker.com -o get-docker.sh

sudo sh get-docker.sh #Al ejecutar el segundo comando, se debe dejar que corra el script.

sudo usermod -aG docker $USER

sudo service docker start
```

Verificamos la instalación utilizando el siguiente comando:

```
docker compose version
```

El Docker recien instalado usa `overlayfs`, pero cAdvisor v0.49.1 requiere `overlay2`. Creamos el archivo de configuracion con las siguientes lineas:

```
sudo nano /etc/docker/daemon.json
```

```json
{
  "storage-driver": "overlay2"
}
```

Reiniciamos el servicio:

```
sudo service docker restart
```

Levantamos el stack:

```
sudo mount --make-rshared /

docker compose up -d --build
```

La solucion funciona porque al volver a montar `/` como `rshared` se permite la propagacion de montajes entre el host y los contenedores, lo cual es justamente lo que necesitaba node-exporter para crear su volumen sin errores.

Tras esto, tendremos el stack levantado.

<img width="528" height="230" alt="Image" src="https://github.com/user-attachments/assets/3ed174f9-c045-4e7f-90de-d196e30765f7" />

Verificamos si estan levantados los contenedores:

```
docker ps
```

## 3. Acceso a las aplicaciones

En nuestro navegador colocamos `localhost` seguido de los siguientes puertos para iniciar las apps:

- Backend: `localhost:8080`
- Frontend: `localhost:3001`
- Grafana: `localhost:3000`
- Prometheus: `localhost:9090`
- cAdvisor: `localhost:9100`

## 4. Generar metricas iniciales

Para iniciar con las metricas, vamos al frontend y presionamos en "saludar API" un par de veces para tener un pequeno registro. Para obtener las métricas colocamos el endpoint /metrics exigido por el protocolo de scraping de Prometheus.
`localhost:8080/metrics`

<img width="837" height="270" alt="Image" src="https://github.com/user-attachments/assets/2b55dbfa-fa87-4749-9e30-9509769b7661" />

## 5. Creacion de Dashboards

### CPU del contenedor de la aplicacion

Colocar la siguiente configuracion:

- Fuente: Prometheus
- Tipo de visualizacion: Time series
- Unit: Percent (0-100)

Colocar el siguiente PromQL query en queries:
```
sum(rate(container_cpu_usage_seconds_total{name="lab-backend"}[1m])) * 100
```
### CPU del host

Colocar la siguiente configuracion:

- Fuente: Prometheus
- Tipo de visualizacion: Time series
- Unit: Percent (0-100)

Colocar el siguiente PromQL query en queries:

```
sum(rate(container_cpu_usage_seconds_total[1m])) * 100
```

### Logs de aplicacion (API + frontend)

- Fuente: Loki
- Tipo de visualizacion: Logs

Colocar el siguiente codigo:

```
{tier="application"} | json

{tier="application"} | json | level="level" #Para los filtros por niveles
```

### Logs de infraestructura

- Fuente: Loki
- Tipo de visualizacion: Logs

Colocar el siguiente codigo:

```
{tier="infrastructure"}
```

## 6. Configuracion de alerta

Durante la creacion de la alerta del CPU del backend, colocar la siguiente query junto a un "IS ABOVE" de 50, un pending period de 30 segundos, y la creacion de una carpeta y etiqueta nombrada `severity = warning`.

```
sum(rate(container_cpu_usage_seconds_total{name="lab-backend"}[1m])) * 100
```

## 7. Cerrar el ciclo de la alarma

Se crea un contact point con integracion de webhook colocando la siguiente url:

```
http://backend:3001/alerts
```

Tras crearlo, debes dirigirte a politicas de notificaciones, editar la politica por defecto y seleccionar el punto de contacto creado anteriormente, actualizando la politica.

Finalmente, debes integrar la alerta creada anteriormente con el punto de contacto creado, en el apartado de configuraciones de notificaciones, editando la alerta.

Para las verificaciones de estas al aumentar la carga, colocar los siguientes comandos dentro de los paneles de logs:

- Dentro del panel de aplicaciones, para recibir las solicitudes:

```
{tier="application"} | json | method = "POST"
```

- Dentro del panel de logs de infraestructura, para recibir las alertas:

```
{tier="infrastructure"} |= "alert"
```
