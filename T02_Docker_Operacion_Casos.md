# T02 · Docker en operación: 8 casos de incidencias

> Continuación de `T01_Docker_Fundamentos.md`. En T01 aprendisteis qué es una imagen, un contenedor y cómo se levanta la pizzería. Aquí **operáis** el sistema: algo falla, lo diagnosticáis y lo arregláis, como en un trabajo real.

**Estado:** borrador. Los pasos marcados **[POR VERIFICAR EN LAB LIMPIO]** hay que probarlos en un laboratorio limpio antes de darlos por buenos.

---

## Cómo usar este documento

- Cada caso dura **15–20 minutos**. Se hacen en orden, pero son independientes: si uno os atasca, pasad al siguiente.
- Cada caso tiene: **Síntoma**, **Objetivo**, **Idea clave**, **Pistas** (3 niveles) y **Cómo sabéis que está resuelto**.
- **Las soluciones están al final del fichero.** Intentad primero con las pistas, de una en una. Abrir la solución sin intentarlo no os enseña nada.
- Trabajáis en la EC2 de vuestro laboratorio, dentro de la carpeta del proyecto.

### Reglas de seguridad (valen para todos los casos)

1. **Nunca** `docker compose down -v` ni `docker system prune -a --volumes`: borran volúmenes y datos.
2. **No toquéis el contenedor del túnel ni su token** salvo que el caso lo diga.
3. **No toquéis la base de datos RDS**: los casos operan solo con contenedores.
4. Antes de editar un fichero de configuración, **haced copia** (`cp fichero fichero.bak`).
5. Para saber el nombre real de cada servicio: `docker compose ps` (usad los mismos ficheros `-f` que en la guía de despliegue). En este documento se escriben `backend`, `frontend-web` y `cloudflared`; si en vuestro compose se llaman distinto, usad vuestros nombres.

### Comandos de consulta que usaréis siempre

```bash
docker compose ps              # estado de los servicios
docker compose logs --tail 50 backend   # últimas 50 líneas del backend
docker ps -a                   # todos los contenedores, también los parados
docker inspect -f '{{.State.ExitCode}}' <contenedor>   # código de salida
```

---

## Caso 1 · El backend se cae

**Síntoma.** La web carga, pero los pedidos y el menú dan error. La app del móvil no muestra pizzas.

**Objetivo.** Provocar la caída de dos formas distintas y entender por qué una se recupera sola y la otra no.

**Idea clave.** La política de reinicio `unless-stopped` reinicia un contenedor que **muere**, pero respeta que **tú lo hayas parado**.

**Qué hacer.**

1. Anotad el estado inicial con `docker compose ps`.
2. Matad el backend con `docker kill`. Esperad 10–20 segundos y mirad el estado.
3. Ahora paradlo con `docker stop`. Esperad lo mismo y mirad el estado.
4. Dejad el sistema funcionando de nuevo.

**Pistas.**

- _Nivel 1:_ para matar y parar necesitáis el nombre del contenedor, no el del servicio. Sacadlo con `docker ps`.
- _Nivel 2:_ comparad `docker compose ps` después de `kill` y después de `stop`. ¿En cuál ha vuelto a estar "Up"?
- _Nivel 3:_ `docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' <contenedor>` os dice la política. Para volver a levantar uno parado a mano: `docker compose up -d backend`.

**Resuelto cuando.** Sabéis explicar con vuestras palabras por qué `kill` se recupera y `stop` no, y el backend vuelve a aparecer como "Up" con la web mostrando el menú.

---

## Caso 2 · Cambias código y no se nota

**Síntoma.** Habéis modificado un texto del frontend (o una respuesta del backend), hacéis `docker compose restart` y en el navegador sigue saliendo lo de antes.

**Objetivo.** Entender la diferencia entre reiniciar un contenedor y reconstruir su imagen.

**Idea clave.** `restart` reinicia **el mismo contenedor con la misma imagen**. El código viejo está dentro de la imagen. Para que el cambio entre hay que **reconstruir la imagen** y recrear el contenedor.

**Qué hacer.**

1. Haced un cambio visible y pequeño (por ejemplo, un texto del frontend). Anotad cuál.
2. Aplicad `docker compose restart <servicio>` y comprobad en el navegador (Ctrl+Shift+R).
3. Aplicad ahora el método correcto y comprobad de nuevo.
4. Deshaced el cambio con Git y volved a aplicar el método correcto para dejarlo como estaba.

**Pistas.**

- _Nivel 1:_ mirad en el `Dockerfile` si el código se copia a la imagen (`COPY`) o se monta desde fuera.
- _Nivel 2:_ existe una opción de `up` que reconstruye imágenes.
- _Nivel 3:_ `docker compose up -d --build <servicio>`. Para deshacer el cambio: `git checkout -- <fichero>` y otra vez `up -d --build`.

**Resuelto cuando.** Veis el cambio en el navegador tras el método correcto, y habéis vuelto al texto original.

---

## Caso 3 · Cambias el `.env` y no pasa nada

**Síntoma.** Editáis una variable en `.env` y reiniciáis el servicio. El servicio sigue con el valor antiguo.

**Objetivo.** Saber cómo se aplica un cambio de variables de entorno.

**Idea clave.** Las variables se **inyectan al crear el contenedor**. `restart` no relee `.env`; `up -d <servicio>` detecta el cambio y **recrea** el contenedor.

**Qué hacer.**

1. Elegid una variable **inofensiva** (por ejemplo, una de nombre o de texto que el backend muestre o escriba en logs). **No cambiéis ninguna contraseña ni el token del túnel.**
2. Copiad el fichero: `cp .env .env.bak`.
3. Cambiad el valor, haced `restart` y comprobad con `docker compose exec <servicio> env` si el servicio ve el valor nuevo.
4. Aplicad el cambio correctamente y volved a comprobar.
5. Restaurad el valor original con la copia y aplicad otra vez.

**Pistas.**

- _Nivel 1:_ `docker compose exec <servicio> env` muestra las variables que ve el contenedor. Filtrad con `| grep NOMBRE`.
- _Nivel 2:_ ¿qué comando recrea un contenedor sin reconstruir la imagen?
- _Nivel 3:_ `docker compose up -d <servicio>`.

**Resuelto cuando.** `env` dentro del contenedor muestra el valor nuevo, y después el original tras restaurar.

> Aviso: no mostréis por pantalla (ni peguéis en el chat) variables con contraseñas o tokens. Elegid una variable que no sea secreta.

---

## Caso 4 · Contraseña de base de datos errónea

**Síntoma.** Tras un cambio, el backend arranca y se para una y otra vez (o queda "Restarting"). La web da error 502 o no carga datos.

**Objetivo.** Leer los logs, identificar un fallo de autenticación con la BD y corregirlo.

**Idea clave.** Si el backend no puede conectar con la base de datos, falla al arrancar; el reinicio automático lo hace entrar en bucle. **Los logs dicen la causa**.

**Qué hacer.** (Trabajad por parejas: uno rompe, otro diagnostica. Después se cambian.)

1. Quien rompe: copia `.env` a `.env.bak` y cambia **solo** el valor de `DB_PASSWORD` por uno incorrecto. Aplica con `docker compose up -d backend`. **No cambia nada en RDS.**
2. Quien diagnostica (sin saber qué se cambió): mira `docker compose ps` y `docker compose logs --tail 50 backend`. Debe decir cuál es la causa y qué variable sospecha.
3. Se restaura el valor correcto desde la copia y se aplica de nuevo.

**Pistas.**

- _Nivel 1:_ buscad en los logs las palabras `password`, `authentication` o `FATAL`.
- _Nivel 2:_ el mensaje habla de la base de datos, no del backend. ¿Quién tiene la contraseña correcta: RDS o el `.env`?
- _Nivel 3:_ restaurad `.env` desde `.env.bak` y aplicad con `docker compose up -d backend` (no `restart`: no relee `.env`).

**Resuelto cuando.** El backend queda "Up" estable, sin reinicios, y la web muestra el menú.

---

## Caso 5 · `nginx.conf` con un error

**Síntoma.** Tras editar la configuración de Nginx, el frontend no arranca o deja de responder.

**Objetivo.** Validar una configuración **antes** de aplicarla, y tener siempre una copia para volver atrás.

**Idea clave.** `nginx -t` comprueba la sintaxis sin tocar el servicio. Y **siempre se hace copia antes de editar**.

**Qué hacer.**

1. Copiad: `cp frontend-web/nginx.conf frontend-web/nginx.conf.bak`.
2. Introducid un error de sintaxis (por ejemplo, quitad un `;` al final de una directiva o una llave `}`).
3. Comprobad la sintaxis **sin romper el servicio**: usad un contenedor temporal con esa configuración, o aplicad y mirad logs. Elegid vosotros, pero explicad por qué es más seguro.
4. Restaurad la copia y aplicad el cambio correctamente.

**Pistas.**

- _Nivel 1:_ el comando para probar la sintaxis de Nginx es `nginx -t`. Se ejecuta _dentro_ de un contenedor de Nginx.
- _Nivel 2:_ con el servicio ya levantado, `docker compose exec frontend-web nginx -t` prueba la configuración que ve ese contenedor. **[POR VERIFICAR EN LAB LIMPIO]** cómo está montado `nginx.conf`: si va dentro de la imagen, hará falta `up -d --build frontend-web` para que el contenedor vea el cambio; si va por bind mount, bastará con recrearlo o recargar.
- _Nivel 3:_ `cp frontend-web/nginx.conf.bak frontend-web/nginx.conf` y aplicar con el mismo método que usasteis para meter el error.

**Resuelto cuando.** `nginx -t` dice `syntax is ok` y `test is successful`, y la web carga.

---

## Caso 6 · El túnel está caído

**Síntoma.** `https://<vuestro-subdominio>/` da un error de Cloudflare (**530** o **1033**) pero, dentro de la EC2, el frontend responde.

**Objetivo.** Distinguir un fallo del túnel de un fallo de la aplicación.

**Idea clave.** Si la web funciona por dentro y falla por fuera, el problema está **entre Cloudflare y vuestro servidor**: el contenedor del túnel, su token o la ruta del túnel.

**Qué hacer.**

1. Comprobad por dentro: `curl -sI http://localhost` desde la EC2 (el frontend responde).
2. Parad **solo** el contenedor del túnel con `docker compose stop cloudflared` (sin tocar su token ni su configuración).
3. Comprobad desde el navegador: ¿qué error sale y con qué código?
4. Mirad `docker compose ps` y los logs del túnel: `docker compose logs --tail 30 cloudflared`.
5. Volved a levantarlo y comprobad que la web vuelve.

**Pistas.**

- _Nivel 1:_ mirad si `cloudflared` aparece "Up" en `docker compose ps`.
- _Nivel 2:_ cuando el contenedor funciona pero el error persiste, la causa suele estar en la **ruta del túnel** en el panel de Cloudflare: debe apuntar a `http://frontend-web:80`, no a `localhost`.
- _Nivel 3:_ `docker compose up -d cloudflared`.

**Resuelto cuando.** Sabéis decir en una frase cómo distinguir "app caída" de "túnel caído", y la web vuelve a cargar por fuera.

---

## Caso 7 · Un despliegue que sale mal y vuelta atrás con Git

**Síntoma.** Tras un cambio en el código, la aplicación deja de funcionar. Hace diez minutos funcionaba.

**Objetivo.** Usar Git para volver a una versión que funcionaba y volver a desplegar.

**Idea clave.** Si todo cambio está en Git, **volver atrás es volver a un commit anterior** y reconstruir. Sin Git, no hay vuelta atrás fiable.

**Qué hacer.**

1. Anotad el commit actual: `git log --oneline -3`.
2. Haced un cambio que rompa algo evidente (por ejemplo, un error de sintaxis en un fichero del frontend), **sin hacer commit** o haciéndolo en una rama de pruebas. Desplegad con `docker compose up -d --build <servicio>`.
3. Comprobad que falla (navegador y logs).
4. Volved atrás con Git (sin `reset --hard` en la rama principal de otros: usad `git checkout -- <fichero>` o `git revert` si hicisteis commit) y reconstruid.
5. Comprobad que vuelve a funcionar.

**Pistas.**

- _Nivel 1:_ `git status` y `git diff` os dicen qué habéis cambiado.
- _Nivel 2:_ si no habéis hecho commit, deshacer un fichero es `git checkout -- <fichero>`. Si habéis hecho commit, `git revert <commit>` crea un commit que lo deshace.
- _Nivel 3:_ después de deshacer, **hay que reconstruir**: `docker compose up -d --build <servicio>`. Deshacer el fichero no cambia la imagen ya construida.

**Resuelto cuando.** `git status` está limpio (o sólo con el revert) y la aplicación funciona otra vez.

---

## Caso 8 · El disco se llena y un contenedor muere por memoria

Este caso tiene **dos partes**.

### Parte A · Espacio en disco

**Síntoma.** Tras muchos despliegues, la EC2 se queda sin espacio o `docker build` falla con "no space left on device".

**Objetivo.** Ver qué ocupa Docker y limpiar **sin borrar datos**.

**Idea clave.** Cada `--build` deja imágenes antiguas sin etiqueta ("dangling"). Se limpian sin riesgo; los volúmenes, **no se tocan**.

**Qué hacer.**

1. `docker system df` para ver cuánto ocupan imágenes, contenedores, volúmenes y caché.
2. `docker image ls` para ver las imágenes y detectar las `<none>`.
3. Limpiad **solo** imágenes sin usar: `docker image prune`.
4. Volved a ejecutar `docker system df` y comparad.

**Pistas.**

- _Nivel 1:_ la columna `RECLAIMABLE` de `docker system df` dice cuánto se puede recuperar.
- _Nivel 2:_ `docker image prune` pide confirmación y solo borra imágenes sin etiqueta ni contenedor.
- _Nivel 3:_ **nunca** `docker system prune -a --volumes` ni `down -v`.

### Parte B · Contenedor que muere por memoria (código 137)

**Síntoma.** Un contenedor se para de golpe con código de salida **137**.

**Objetivo.** Reproducir un fallo de memoria **en un contenedor de prueba aparte**, nunca en los de la pizzería.

**Idea clave.** 137 = 128 + 9 (señal KILL). Con un límite de memoria muy pequeño, el kernel mata el proceso (OOM). Se reproduce con un contenedor desechable.

**Qué hacer.** **[POR VERIFICAR EN LAB LIMPIO]**

1. Lanzad un contenedor de prueba con poca memoria y que intente usar más. Por ejemplo, con la imagen `alpine`, un límite de `--memory 16m` y un proceso que reserve más (ver pista 3).
2. Cuando muera, mirad el código de salida con `docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}' <contenedor>`.
3. Borrad el contenedor de prueba.

**Pistas.**

- _Nivel 1:_ `docker run --name prueba-oom --memory 16m alpine ...` limita la memoria a 16 MB.
- _Nivel 2:_ hace falta un comando que consuma memoria. Algo sencillo con `sh` o `dd` sirve.
- _Nivel 3:_ ver la solución al final. Y recordad: es un contenedor **aparte**; `docker rm prueba-oom` al terminar.

**Resuelto cuando.** Habéis visto `137` y `OOMKilled true` en un contenedor de prueba, lo habéis eliminado, y `docker system df` no muestra cambios inesperados.

---

# Tabla resumen

| Caso | Qué falla                 | Comando clave                               | Error típico                      |
| ---- | ------------------------- | ------------------------------------------- | --------------------------------- |
| 1    | Contenedor caído          | `docker compose ps`, `docker kill` / `stop` | Creer que `stop` se recupera solo |
| 2    | Código cambiado, no se ve | `up -d --build <servicio>`                  | Usar `restart`                    |
| 3    | `.env` cambiado, no se ve | `up -d <servicio>`                          | Usar `restart`                    |
| 4    | Contraseña BD errónea     | `logs --tail 50 backend`                    | No leer los logs                  |
| 5    | `nginx.conf` roto         | `nginx -t`, copia `.bak`                    | Editar sin copia                  |
| 6    | Túnel caído (530/1033)    | `logs cloudflared`, ruta del túnel          | Buscar el fallo en la app         |
| 7    | Despliegue roto           | `git checkout` / `git revert` + `--build`   | Deshacer y no reconstruir         |
| 8    | Disco lleno / OOM 137     | `docker system df`, `image prune`           | `prune -a --volumes`              |

# Preguntas de repaso

1. ¿Qué diferencia hay entre `docker kill` y `docker stop` respecto a la política `unless-stopped`?
2. ¿Por qué `restart` no aplica cambios de código ni de `.env`? ¿Qué se usa en cada caso?
3. Si la web funciona dentro de la EC2 pero falla desde fuera, ¿dónde buscáis primero?
4. ¿Por qué es importante tener todo en Git antes de desplegar?
5. ¿Qué comandos de limpieza son seguros y cuáles pueden destruir datos?
6. ¿Qué significa el código de salida 137?

---

# Soluciones

> No las abráis hasta haber probado las pistas.

## Solución caso 1

```bash
docker compose ps
docker kill <contenedor_backend>
sleep 15 && docker compose ps       # vuelve a "Up": la política lo reinicia
docker stop <contenedor_backend>
sleep 15 && docker compose ps       # sigue parado: lo paraste tú
docker compose up -d backend        # lo levantas de nuevo
```

`unless-stopped` reinicia lo que muere, pero no lo que el usuario ha parado de forma explícita.

## Solución caso 2

```bash
docker compose up -d --build <servicio>
```

El código está copiado dentro de la imagen; `restart` no la reconstruye. Para deshacer: `git checkout -- <fichero>` y volver a `up -d --build`.

## Solución caso 3

```bash
cp .env .env.bak
# editar .env
docker compose restart <servicio>                      # NO aplica el cambio
docker compose exec <servicio> env | grep NOMBRE       # sigue el valor viejo
docker compose up -d <servicio>                        # recrea el contenedor: sí aplica
docker compose exec <servicio> env | grep NOMBRE       # valor nuevo
cp .env.bak .env && docker compose up -d <servicio>    # restaurar
```

## Solución caso 4

```bash
docker compose ps                         # backend en "Restarting"
docker compose logs --tail 50 backend     # mensaje de autenticación fallida con la BD
cp .env.bak .env
docker compose up -d backend              # recrea con la contraseña correcta
```

La causa está en el log: el backend intenta conectar a la BD con una contraseña que no coincide. `restart` no habría servido, porque no relee `.env`.

## Solución caso 5

```bash
cp frontend-web/nginx.conf frontend-web/nginx.conf.bak
# introducir el error (p. ej. quitar un ';')
docker compose exec frontend-web nginx -t       # muestra el error y la línea
cp frontend-web/nginx.conf.bak frontend-web/nginx.conf
docker compose up -d --build frontend-web       # si nginx.conf va dentro de la imagen
docker compose exec frontend-web nginx -t       # "syntax is ok / test is successful"
```

Validar antes de aplicar evita dejar el frontend caído. Si en vuestro compose `nginx.conf` va por bind mount, no hace falta `--build`; bastará con recrear el servicio. **[POR VERIFICAR EN LAB LIMPIO]**

## Solución caso 6

```bash
curl -sI http://localhost                      # el frontend responde por dentro
docker compose stop cloudflared                # fuera: error 530 / 1033
docker compose logs --tail 30 cloudflared
docker compose up -d cloudflared               # la web vuelve
```

Regla práctica: **por dentro bien y por fuera mal = problema del túnel** (contenedor parado, token, o ruta del túnel que debe ser `http://frontend-web:80`).

## Solución caso 7

```bash
git log --oneline -3
# cambio que rompe algo
docker compose up -d --build <servicio>        # falla
git status && git diff
git checkout -- <fichero>                      # sin commit
# o, si hubo commit:  git revert <commit>
docker compose up -d --build <servicio>        # hay que reconstruir
```

Deshacer el fichero no cambia la imagen ya construida: sin `--build` seguiría el código roto.

## Solución caso 8

Parte A:

```bash
docker system df
docker image ls
docker image prune          # confirma con 'y'; solo imágenes sin etiqueta ni uso
docker system df
```

Nunca `docker system prune -a --volumes` ni `docker compose down -v`: borran volúmenes (datos).

Parte B **[POR VERIFICAR EN LAB LIMPIO]**:

```bash
docker run --name prueba-oom --memory 16m alpine sh -c 'a=x; while true; do a="$a$a"; done'
docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}' prueba-oom   # 137 true
docker rm prueba-oom
```

137 = 128 + 9 (SIGKILL). El kernel mata el proceso al superar el límite de memoria.

---

## Notas para el profesorado

- Casos pensados para 15–20 min; en una sesión de 2 h caben 5–6 con puesta en común.
- Los casos 4 y 7 funcionan muy bien por parejas ("uno rompe, otro diagnostica").
- Ningún caso necesita scripts nuevos ni tocar `backend/src`.
- Antes de usarlo con alumnos: pasar los casos 5 y 8B en un laboratorio limpio y anotar los nombres reales de servicios y contenedores.
