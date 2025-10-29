---
layout: single
title: Hackeando el 80% de los Buffets de Sushi - Vulnerabilidades Críticas en Sistemas de Gestión
excerpt: Descubrimiento de vulnerabilidades críticas en la infraestructura compartida de múltiples restaurantes de sushi que exponen datos sensibles de clientes y permiten manipular operaciones.
date: 2025-10-29
classes: wide
header:
  teaser: /assets/images/shushi/shushi.png
  teaser_home_page: true
  icon:/assets/images/shushi/shushi.png
categories:
  - Review
  - Infosec
  - Vulns
  - Hacking
  - Pentesting
tags:
  - Pentesting Web
  - API Security
  - Swagger
  - Broken Access Control
  - OWASP Top 10
---

# 🔍 El Descubrimiento Inicial

Estaba disfrutando de mi comida cuando decidí echar un vistazo a la URL del sistema de pedidos del restaurante. Una vez en casa, mi curiosidad me llevó a investigar más a fondo. Comencé con algo básico: **enumeración de subdominios**.

Para mi sorpresa, descubrí algo fascinante: **todos los restaurantes de diferentes cadenas estaban bajo el mismo dominio**. Pero no solo eso, encontré múltiples subdominios con el patrón `xxapi.*` que llamaron inmediatamente mi atención.

![Subdominios API encontrados](/assets/images/shushi/apisubdomains.png)

## 🚨 Swagger Expuesto: La Puerta de Entrada

Al acceder a estos subdominios de API, me encontré con algo que nunca debería estar público en producción: **Swagger UI completamente expuesto** o al menos en este contexto. 

> Para los que no lo saben, Swagger es una herramienta de documentación de APIs extremadamente útil para desarrolladores... pero que nunca debería estar accesible públicamente en un entorno de producción, especialmente sin autenticación. (obviamente depende el contexto)

![Swagger UI expuesto](/assets/images/shushi/Swagger.png)

Y aquí es donde las cosas se pusieron realmente interesantes.

## ¿Qué se podía hacer sin autenticación?

### 1. 📊 Listar Todas las Mesas y Sus Datos Sensibles

El endpoint `/admins/Table/GetTables` permitía obtener información completa de todas las mesas del restaurante:

```
POST /admins/Table/GetTables
```

![Respuesta del endpoint GetTables](/assets/images/shushi/GetTables.png)

**Información expuesta:**
- Identificadores únicos de mesas y áreas
- Nombres de salones y ubicaciones
- Códigos QR de las mesas
- **Gastos totales de cada mesa en tiempo real**
- **Productos pedidos por cada cliente**
- **Métodos de pago utilizados**
- Información de cuentas y contraseñas

Imagina las implicaciones: un atacante podría saber exactamente cuánto está gastando cada mesa, qué están comiendo, y planificar ataques dirigidos.

#### 2. 📱 Manipular Mensajes en las Tablets

El endpoint permitía modificar los mensajes que aparecen en las tablets de los clientes. Esto podría usarse para:
- Phishing dirigido a clientes en el restaurante
- Crear pánico o confusión
- Redirigir a sitios maliciosos
- Mostrar ofertas falsas

#### 3. 💸 Cerrar Mesas (¡Comer Gratis!)

Uno de los hallazgos más críticos: era posible cerrar mesas remotamente sin pagar. Un atacante podría:
1. Sentarse en una mesa
2. Pedir todo lo que quisiera
3. Cerrar la cuenta remotamente usando la API
4. Irse sin pagar

Esto representa un **riesgo financiero directo** para los establecimientos.

#### 4. 💰 Modificar Precios y Menús

Los endpoints expuestos también permitían:
- Cambiar precios de productos
- Modificar descripciones del menú
- Añadir o eliminar productos
- Alterar toda la carta del restaurante

## 🗺️ Alcance del Problema

Lo más preocupante de todo esto es el alcance. Durante mi investigación descubrí que:

- **Múltiples restaurantes** utilizan la misma infraestructura
- **Todos compartían las mismas vulnerabilidades**
- La API era accesible desde cualquier lugar con conexión a Internet
- No había sistemas de detección de intrusos aparentes
- No existían límites de tasa (rate limiting)

Esto significa que un atacante podría:
- Extraer datos de todos los restaurantes de forma masiva
- Automatizar ataques contra múltiples establecimientos
- Causar pérdidas financieras significativas
- Comprometer la privacidad de miles de clientes


## 🛡️ Divulgación Responsable

**Es importante destacar que esta vulnerabilidad ha sido reportada de forma responsable** al proveedor del software. Como investigador de seguridad ético, mi objetivo nunca es causar daño, sino ayudar a mejorar la seguridad de los sistemas.

## 💭 Reflexiones Finales

Este caso es un **ejemplo perfecto de cómo la comodidad del desarrollo puede comprometer gravemente la seguridad**. Dejar Swagger expuesto en producción es como dejar el plano de tu casa en la puerta de entrada con todas las cerraduras marcadas.

### Lecciones Aprendidas:

1. **La seguridad debe ser prioritaria desde el diseño**: No es algo que se añade después.
2. **La documentación de APIs es para desarrollo, no para producción**: Herramientas como Swagger deben estar deshabilitadas en entornos de producción.
3. **Cada endpoint debe tener autenticación**: Especialmente aquellos que manejan operaciones sensibles.
4. **El principio de defensa en profundidad es crucial**: No basta con una sola capa de seguridad.

## 🚀 Conclusión

Lo que comenzó como una simple curiosidad durante una comida de sushi se convirtió en el descubrimiento de vulnerabilidades críticas que afectan a una parte significativa de la industria. Este hallazgo demuestra que:

- **Los sistemas POS (Point of Sale) modernos** son tan vulnerables como cualquier otra aplicación web
- **La seguridad API** sigue siendo una asignatura pendiente en muchas organizaciones
- **La divulgación responsable** es esencial para mejorar la seguridad de todos

Y recordad: **la próxima vez que pidan sushi desde una tablet, piensen en todo lo que hay detrás de esa interfaz simple**. 🍣🔒

---

*Nota: Todas las imágenes han sido editadas para proteger la identidad de los establecimientos afectados. La información específica que podría identificar a las empresas ha sido redactada.*

*Este artículo se publica con fines educativos y de concienciación sobre seguridad. No se incluye información que facilite la explotación de estas vulnerabilidades.*

