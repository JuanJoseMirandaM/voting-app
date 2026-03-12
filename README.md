# Proyecto Final: Docker — Voting App

Aplicación de votación distribuida. El objetivo es **crear los Dockerfiles** para cada servicio y un **compose.yml** que orqueste toda la aplicación.

---

## Objetivo del proyecto

Al finalizar debes tener:

1. **Dockerfile** para el servicio **Vote** (Python/Flask).
2. **Dockerfile** para el servicio **Worker** (Node.js).
3. **Dockerfile** para el servicio **Result** (Node.js).
4. **compose.yml** en la raíz del repositorio que levante:
   - Los tres servicios anteriores.
   - Redis.
   - PostgreSQL.

Con `docker compose up --build` la aplicación debe quedar funcionando y accesible.

---

## Arquitectura de la aplicación
![](./docs/5.png)

| Servicio    | Descripción |
|------------|-------------|
| **Vote**   | Web en Flask para votar (opción A o B). Envía votos a Redis. |
| **Worker** | Proceso Node.js que lee votos de Redis y los persiste en PostgreSQL. |
| **Result** | Web Node.js que lee votos de PostgreSQL y los muestra en tiempo real (WebSockets). |
| **Redis**  | Cola de mensajes entre Vote y Worker. |
| **PostgreSQL** | Base de datos donde se guardan los votos. |

---

## Estructura del repositorio

Coloca los Dockerfiles y el compose como se indica (los servicios están dentro de la carpeta `voting-app/`):

```text
.
├── voting-app/
│   ├── vote/                    # Servicio Vote (Flask)
│   │   ├── app.py
│   │   ├── requirements.txt
│   │   ├── templates/
│   │   └── Dockerfile           ← CREAR
│   ├── worker/                  # Servicio Worker (Node.js)
│   │   ├── main.js
│   │   ├── package.json
│   │   └── Dockerfile           ← CREAR
│   ├── result/                  # Servicio Result (Node.js)
│   │   ├── main.js
│   │   ├── package.json
│   │   ├── views/               # HTML, CSS, JS del front
│   │   └── Dockerfile           ← CREAR
│   └── ...
├── compose.yml           ← CREAR (en la raíz del repo)
└── README.md
```

No se incluyen Dockerfile ni compose; debes crearlos tú. En `compose.yml` usa como build context las rutas `voting-app/vote`, `voting-app/worker` y `voting-app/result`, o coloca el `compose.yml` dentro de `voting-app/` y usa `./vote`, `./worker`, `./result`.

---

## Servicios y variables de entorno

Usa estas referencias al escribir los Dockerfiles y el `compose.yml`.

### Vote (Flask)

| Variable        | Descripción        | Valor por defecto |
|----------------|--------------------|-------------------|
| `REDIS_HOST`   | Host de Redis      | `redis`           |
| `OPTION_A`     | Texto opción A     | `Cats`            |
| `OPTION_B`     | Texto opción B     | `Dogs`            |
| `DATABASE_HOST`| Host PostgreSQL    | `database`        |
| `DATABASE_USER`| Usuario PostgreSQL | `postgres`        |
| `DATABASE_PASSWORD` | Contraseña   | `postgres`        |
| `DATABASE_NAME`| Nombre de la BD    | `votes`           |

- Puerto interno: **80** (o el que use la app en `app.py`).
- Comando típico: servir con **gunicorn** (ya está en `requirements.txt`). Ejemplo de `CMD` en el Dockerfile:

  ```dockerfile
  CMD ["gunicorn", "--bind", "0.0.0.0:80", "app:app"]
  ```

### Worker (Node.js)

| Variable        | Descripción        | Valor por defecto |
|----------------|--------------------|-------------------|
| `REDIS_HOST`   | Host de Redis      | `redis`           |
| `DATABASE_HOST`| Host PostgreSQL    | `database`        |
| `DATABASE_USER`| Usuario PostgreSQL | `postgres`        |
| `DATABASE_PASSWORD` | Contraseña   | `postgres`        |
| `DATABASE_NAME`| Nombre de la BD    | `votes`           |

- Punto de entrada: `node main.js` (o `npm start`).
- Versión Node recomendada: **20.x** (ver `worker/.nvmrc` si existe).

### Result (Node.js)

| Variable        | Descripción        | Valor por defecto |
|----------------|--------------------|-------------------|
| `DATABASE_HOST`| Host PostgreSQL    | `database`        |
| `DATABASE_USER`| Usuario PostgreSQL | `postgres`        |
| `DATABASE_PASSWORD` | Contraseña   | `postgres`        |
| `DATABASE_NAME`| Nombre de la BD    | `votes`           |
| `APP_PORT`     | Puerto HTTP        | `3000`            |

- Punto de entrada: `node main.js` (o `npm start`).
- Versión Node recomendada: **20.x** (ver `result/.nvmrc` si existe).
- Servir la carpeta **views** como estáticos (la app ya usa rutas relativas a `views/`).

### Redis

- Imagen oficial: `redis:6` o superior.
- Puerto: **6379**.

### PostgreSQL

- Imagen oficial: `postgres:15` o superior.
- Variables: `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` (por ejemplo `postgres`, `postgres`, `votes`).
- Puerto: **5432**.
- El **Worker** crea la tabla `votes` si no existe; puedes usar un volumen para persistir datos.

---

## Orden de arranque y redes

- **PostgreSQL** y **Redis** deben estar disponibles antes de que arranquen Vote, Worker y Result.
- En `compose.yml` usa `depends_on` para ordenar los servicios.
- Todos los servicios deben estar en la **misma red** para resolverse por nombre (`redis`, `database`, etc.).

---

## Cómo probar tu solución

1. En la raíz del repo:

   ```bash
   docker compose up --build
   ```

2. Cuando todos los contenedores estén en ejecución:
   - **Votar:** abrir en el navegador el puerto mapeado al servicio **Vote** (ej. `http://localhost:5000` si mapeaste 5000:80).
   - **Ver resultados:** abrir el puerto mapeado al servicio **Result** (ej. `http://localhost:5001` si mapeaste 5001:3000).

3. Debe ser posible votar, ver la página de resultados actualizarse y que los votos persistan al reiniciar los contenedores (si configuraste volumen para PostgreSQL).

![](./docs/2.png)
![](./docs/1.png)

---

## Resumen de entregables

| Entregable           | Ubicación                  | Descripción |
|----------------------|----------------------------|-------------|
| Dockerfile Vote      | `voting-app/vote/Dockerfile`  | Imagen Flask con dependencias y comando gunicorn. |
| Dockerfile Worker    | `voting-app/worker/Dockerfile`| Imagen Node.js con `npm install` y `node main.js`. |
| Dockerfile Result    | `voting-app/result/Dockerfile`| Imagen Node.js con `npm install`, `node main.js` y archivos `views/`. |
| compose.yml   | Raíz del repo              | Definición de los 5 servicios, variables de entorno, puertos y `depends_on`. |

---

## Versiones recomendadas

| Componente   | Versión        |
|-------------|-----------------|
| Python (Vote) | 3.11+         |
| Node (Worker / Result) | 20.x LTS |
| Redis       | 6.x             |
| PostgreSQL  | 15.x            |

---

## Referencias

- [Docker — Documentación](https://docs.docker.com/)
- [Docker Compose](https://docs.docker.com/compose/)
- [Dockerfile reference](https://docs.docker.com/engine/reference/builder/)

---

*Proyecto basado en el [Docker Example Voting App](https://github.com/dockersamples/example-voting-app), adaptado para proyecto final de Docker.*
