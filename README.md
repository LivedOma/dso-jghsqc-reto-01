# Reto 1 — CI con GitHub Actions → imagen Docker → Docker Hub

Microservicio Java (Spring Boot 3 / Java 17) con pipeline de integración continua que compila el JAR, construye la imagen Docker y la publica en Docker Hub con etiquetas trazables.

---

## 1. Paso a paso

### 1.1 Crear el repositorio y subir el código

```bash
cd demo-micro
git init
git add .
git commit -m "feat: microservicio demo + Dockerfile + workflow de CI"
git branch -M main
git remote add origin https://github.com/<TU_USUARIO>/demo-micro.git
git push -u origin main
```

### 1.2 Crear el token de Docker Hub

1. Docker Hub → **Account Settings → Personal access tokens → Generate new token**
2. Permisos: **Read & Write**
3. Copiar el token (solo se muestra una vez).

> Se usa un token, **no la contraseña**: es revocable y de permisos acotados.

### 1.3 Configurar los secrets en GitHub

Repositorio → **Settings → Secrets and variables → Actions → New repository secret**

| Nombre | Valor |
|---|---|
| `DOCKERHUB_USERNAME` | tu usuario de Docker Hub |
| `DOCKERHUB_TOKEN` | el token generado en el paso anterior |

### 1.4 Disparar el pipeline

```bash
# Opción A: por commit a main
git commit --allow-empty -m "ci: primera ejecución del pipeline"
git push

# Opción B: por tag semántico
git tag v1.0.0
git push origin v1.0.0
```

---

## 2. Verificación (evidencias del reto)

**En GitHub → Actions:** el workflow *CI - Build & Push Docker* en verde, con los pasos Checkout → Java → Tests → Build jar → Buildx → Login → Metadata → Build & push.

**En Docker Hub:** el repositorio `<usuario>/demo-micro` con las etiquetas:

| Etiqueta | Origen |
|---|---|
| `main` | `type=ref,event=branch` |
| `v1.0.0` | `type=ref,event=tag` |
| `sha-a1b2c3d` | `type=sha` |
| `latest` | solo desde la rama por defecto |

**Prueba local de la imagen publicada:**

```bash
docker pull <usuario>/demo-micro:main
docker run -d --name demo-micro -p 8080:8080 -e APP_VERSION=1.0.0 <usuario>/demo-micro:main

curl http://localhost:8080/
curl http://localhost:8080/version
curl http://localhost:8080/actuator/health

docker logs demo-micro
docker rm -f demo-micro
```

Confirmar que **no corre como root**:

```bash
docker run --rm <usuario>/demo-micro:main whoami   # -> appuser
```

---

## 3. Diferencias respecto al workflow del enunciado

| Cambio | Por qué |
|---|---|
| Paso `mvn -B test` antes del build | El enunciado usa `-DskipTests` en todo el flujo: la imagen se publicaría aunque las pruebas fallen. Se ejecutan primero y el `package` posterior ya puede saltarlas sin repetir trabajo. |
| `push: ${{ github.event_name != 'pull_request' }}` | El workflow original tiene `push: true` fijo, pero también se dispara en `pull_request`: publicaría imágenes desde ramas sin revisar. En PR solo se valida que construya. |
| `login` condicionado a que no sea PR | Los secrets no están disponibles en PRs desde forks; el paso fallaría. |
| `type=raw,value=latest` en la rama por defecto | `latest` es la etiqueta que la gente usa por defecto al hacer pull. |
| Multi-stage de metadatos + usuario `appuser` | Regla del curso: el contenedor no debe ejecutarse como root. |
| `HEALTHCHECK` + actuator | Prepara la imagen para las probes de Kubernetes de las sesiones siguientes. |
| `cache: maven` y `cache-from/to: gha` | Reduce el tiempo del pipeline reutilizando el repositorio M2 y las capas de la imagen. |

---

## 4. Errores frecuentes

| Síntoma | Causa / solución |
|---|---|
| `denied: requested access to the resource is denied` | El nombre de la imagen no coincide con tu usuario de Docker Hub, o el token es de solo lectura. |
| `COPY failed: no source files were found` | El `package` de Maven no corrió antes del build, o `finalName` cambió la ruta del JAR. |
| `Error: Username and password required` | Los secrets están mal nombrados o se crearon a nivel de *environment* en vez de *repository*. |
| El tag `v1.0.0` no aparece en Docker Hub | El push del tag es un evento aparte: `git push origin v1.0.0`. |
| El workflow no se ejecuta | El archivo debe estar exactamente en `.github/workflows/` y con extensión `.yml`. |

---

## 5. Extensiones naturales (siguientes sesiones del curso)

- **SAST:** paso de SonarQube después de los tests + espera del Quality Gate.
- **Escaneo de imagen:** Trivy sobre la imagen construida antes del push, fallando en severidad `CRITICAL`.
- **CD:** job adicional que despliega el tag recién publicado en el clúster de Kubernetes.
