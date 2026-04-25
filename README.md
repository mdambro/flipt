# Flipt Feature Flags Service

Este repositorio contiene la infraestructura y configuración para levantar [Flipt](https://flipt.io/) v2, un servicio open-source para la gestión de Feature Flags (Banderas de Funcionalidades).

Esta implementación está configurada para utilizar el enfoque **GitOps** de Flipt v2. Esto significa que **no utiliza una base de datos relacional pesada**; en su lugar, utiliza un repositorio de Git (ej: GitHub) como única fuente de verdad para almacenar el estado de las flags.

## 🚀 Características principales
- **Interfaz Gráfica Integrada:** Crea, edita y gestiona flags visualmente.
- **GitOps Nativo:** Cualquier cambio realizado en la UI se commitea automáticamente a tu repositorio Git.
- **Sincronización Bidireccional:** Cualquier cambio hecho manualmente en el archivo `features.yml` en Git será detectado y actualizado en Flipt (cada 1 minuto).
- **Liviano:** Al delegar el almacenamiento a Git y almacenamiento local en disco, el consumo de recursos (CPU/RAM) es mínimo.

---

## 📂 Estructura del Proyecto

```
.
├── docker-compose/
│   ├── .env               # Variables de entorno (¡Debes editar este archivo!)
│   └── docker-compose.yml # Definición de los servicios Docker
├── files/
│   └── config.yml         # Configuración central de Flipt
└── README.md              # Este archivo
```

---

## 🛠️ Cómo desplegar el servicio

Sigue estos sencillos pasos para levantar el servidor de Flipt en tu máquina local o VPS.

### 1. Configurar Variables de Entorno y Token de GitHub
Antes de levantar el servicio, necesitas configurar el repositorio Git donde se guardarán las flags y un token de acceso para que Flipt pueda sincronizarlas.

1. Abre el archivo `docker-compose/.env`.
2. Edita las variables relacionadas a Git:
   - `FLIPT_GIT_REPO`: La URL de tu repositorio (ej: `https://github.com/tu-usuario/flags4flipt.git`).
   - `FLIPT_GIT_TOKEN`: Un *Fine-grained Personal Access Token* de GitHub.

> **¿Cómo generar el Token?**
> Ve a GitHub -> Settings -> Developer settings -> Personal access tokens -> Fine-grained tokens. Crea uno que aplique **solo a tu repositorio de flags** y que tenga permisos de **Read and write** exclusivamente en la sección **Contents**.

### 2. Levantar el servicio con Docker
Una vez configurado el `.env`, abre una terminal, navega a la carpeta de `docker-compose` y levanta el servicio en segundo plano:

```bash
cd docker-compose
docker compose up -d
```

### 3. Acceder a la Interfaz Gráfica (UI)
Por defecto, Flipt expone la interfaz web en el puerto `8080`.
Si estás corriendo esto localmente, simplemente abre tu navegador y visita:

👉 **[http://localhost:8080](http://localhost:8080)**

*(Nota: Si usas un VPS, puedes acceder a través de tu proxy inverso apuntando a este contenedor o abriendo el puerto 8080 en el firewall y accediendo a la IP pública).*

---

## 💡 Cómo usar Flipt con GitOps

1. Entra a la interfaz de Flipt.
2. Crea una nueva Flag (Ej: `nueva-interfaz-de-pagos`).
3. Al darle a **Guardar**, Flipt escribirá un archivo YAML en su almacenamiento local y automáticamente hará un `git push` a la rama `main` de tu repositorio de GitHub.
4. Si quieres, ve a tu repositorio en GitHub y verás un nuevo archivo creado con la configuración declarativa de la flag. ¡Todo tu equipo puede auditar los cambios fácilmente!

---

## 💻 Ejemplo de consumo con Fastify

A continuación se muestra un ejemplo básico de cómo consultar el estado de una flag (de tipo booleana) desde un backend construido con [Fastify](https://fastify.dev/) utilizando el SDK oficial de Flipt.

Primero, instala las dependencias necesarias:

```bash
npm install @flipt-io/flipt fastify
```

Luego, en tu código:

```javascript
const Fastify = require('fastify');
const { FliptApiClient } = require('@flipt-io/flipt');

const fastify = Fastify({ logger: true });

// Configurar el cliente apuntando a la URL del servicio Flipt
// Nota: 'environment' recibe la URL base de tu servidor Flipt.
const flipt = new FliptApiClient({ environment: 'http://localhost:8080' });

fastify.get('/check-feature', async (request, reply) => {
  try {
    // Consultar el estado de la flag
    const response = await flipt.evaluation.boolean({
      namespaceKey: 'default',
      flagKey: 'nueva-interfaz-de-pagos',
      entityId: 'user-123', // ID único para el usuario/entidad que realiza la petición
      context: {
        isPremium: "true" // Opcional: contexto extra para reglas de segmentación
      }
    });

    if (response.enabled) {
      return { status: "success", message: "🚀 La nueva interfaz está ACTIVA para este usuario." };
    } else {
      return { status: "success", message: "🐢 La nueva interfaz está DESACTIVADA (Mostrando versión clásica)." };
    }
  } catch (err) {
    fastify.log.error(err);
    reply.status(500).send({ error: "Hubo un error al comunicarse con Flipt." });
  }
});

fastify.listen({ port: 3000 }, (err, address) => {
  if (err) throw err;
  console.log(`Server listening on ${address}`);
});
```
