---
layout: single
title: "Firebase Expuesto: Cuando las Configuraciones Inseguras Comprometen Plataformas Educativas"
excerpt: Análisis de vulnerabilidades críticas en configuraciones de Firebase que permiten manipulación de datos de usuarios, exposición de información personal y acceso no autorizado a sistemas de puntuación.
date: 2025-02-13
classes: wide
header:
  teaser: /assets/images/firebase-exposed/logo.jpeg
  teaser_home_page: true
  icon: /assets/images/firebase-exposed/logo.jpeg
categories:
  - infosec
  - vulnerability
  - firebase
tags:  
  - Firebase
  - NoSQL Security
  - Firestore
  - Web Security
  - Configuration Error
  - Database Security
---

## Introducción: El Lado Oscuro de Firebase

**¡Hola a todos!** Hoy vamos a analizar un problema muy común pero peligroso en el mundo del desarrollo web: **configuraciones inseguras de Firebase**. Durante una investigación de seguridad, descubrí múltiples vulnerabilidades en una plataforma educativa que utiliza Firebase como backend, exponiendo datos de usuarios y permitiendo manipulación no autorizada del sistema.

![Firebase Security](/assets/images/firebase-exposed/logo.jpeg)

### ¿Qué es Firebase y Por Qué es Popular?

Firebase es la plataforma de desarrollo de aplicaciones de Google que ofrece una **base de datos NoSQL (Firestore)**, **autenticación**, **hosting** y muchos otros servicios. Su popularidad se debe a la facilidad de implementación, pero esta simplicidad a menudo lleva a configuraciones de seguridad deficientes.

## El Caso: Plataforma Educativa Comprometida

### Contexto del Descubrimiento

La plataforma analizada es un sitio educativo que incluye:
- **Sistema de cursos** online
- **Comunidad de usuarios** con sistema de puntos
- **Rankings** basados en actividad
- **Perfiles de usuario** con datos personales

### Vulnerabilidad 1: Exposición de Credenciales de Firebase

#### **El Problema**
Las credenciales completas de Firebase están **expuestas en el código JavaScript del cliente**, accesibles desde cualquier navegador.

#### **Archivo Comprometido**
```
GET /assets/index-B1xorCBP.js HTTP/2
```

#### **Credenciales Expuestas**
```javascript
const firebaseConfig = {
    apiKey: "AIzaSyB4pyBl9qQcHfPXXXXXXX",
    authDomain: "whatsapp-XXXXXX.firebaseapp.com",
    projectId: "whatsapp-XXXXXX",
    storageBucket: "whatsapp-XXXXX",
    messagingSenderId: "XXXXX",
    appId: "1:XXXXS:web:XXXXXXX"
};
```

#### **¿Por Qué es Problemático?**
Aunque Google dice que estas credenciales pueden ser públicas, **el verdadero problema son las reglas de seguridad** mal configuradas que permiten acceso no autorizado.

![Firebase Config Exposed](/assets/images/firebase-exposed/fire1.png)

### Vulnerabilidad 2: Reglas de Firestore Inseguras

#### **Acceso Directo a Colecciones**
La colección `ranking` tiene permisos que permiten **lectura y escritura no autorizadas**:

```javascript
// Lectura no autorizada
Getting [DOC_ID] from ranking

// Escritura no autorizada  
Updated fields(Doc ID: [DOC_ID])
{"points":4510,"number":"XXXXX","name":"XXXX"}
```

#### **Impacto Real**
Los usuarios pueden:
- **Leer datos** de otros usuarios
- **Modificar sus propios puntos** arbitrariamente
- **Manipular rankings** sin restricciones
- **Acceder a información personal** de otros usuarios

### Vulnerabilidad 3: Exposición de Datos Personales

#### **Información Comprometida**
Los documentos contienen datos sensibles parcialmente expuestos:

```json
{
    "points": 4510,
    "number": "XXXXX",  // Parte del número de teléfono
    "name": "XXXX"
}
```

#### **Datos Expuestos**
- **Partes de números de teléfono** de usuarios
- **Nombres de usuario** reales
- **Puntuaciones y actividad** de la comunidad
- **IDs de documentos** que pueden revelar patrones


## Demostración Práctica

### Acceso Desde la Consola de Firebase

Con las credenciales expuestas, cualquier persona puede:

1. **Acceder a Firebase Console** con el Project ID
2. **Conectarse a Firestore** directamente
3. **Leer colecciones** sin autenticación
4. **Modificar documentos** si los permisos lo permiten

### Manipulación de Puntos

```javascript
// Desde la consola del navegador
const db = firebase.firestore();

// Leer datos de otros usuarios
db.collection('ranking').get().then((snapshot) => {
    snapshot.forEach((doc) => {
        console.log(doc.id, doc.data());
    });
});

// Modificar mis propios puntos
db.collection('ranking').doc('[MY_DOC_ID]').update({
    points: 999999
});
```

![Data Exposure](/assets/images/firebase-exposed/fire2.png)

## Riesgos e Impacto

### 🎯 **Para los Usuarios**

#### **Privacidad Comprometida**
- **Exposición de números de teléfono** parciales
- **Nombres reales** visibles públicamente
- **Actividad y progreso** accesible sin autorización

#### **Manipulación de Sistema**
- **Rankings falsos** que afectan la credibilidad
- **Competencia desleal** en el sistema de puntos
- **Pérdida de motivación** por rankings manipulados

### 🏢 **Para la Plataforma**

#### **Pérdida de Integridad**
- **Sistema de gamificación** completamente comprometido
- **Confianza de usuarios** dañada
- **Métricas falsas** para análisis de negocio

#### **Riesgos Legales**
- **Incumplimiento de GDPR** por exposición de datos
- **Responsabilidad legal** por filtración de información
- **Posibles sanciones** de autoridades de protección de datos


## Configuraciones Inseguras Comunes

```javascript
// NUNCA hagas esto
allow read, write: if true;  // ¡PELIGROSO!

// Reglas débiles
allow read, write: if request.auth != null;  // Insuficiente
```

## Recomendaciones de Seguridad

### 🛡️ **Configuración Segura de Firestore**

#### **Reglas de Seguridad Robustas**

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Solo el propietario puede leer/escribir sus datos
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
    
    // Rankings solo lectura para autenticados
    match /ranking/{docId} {
      allow read: if request.auth != null;
      allow write: if false;  // Solo por Cloud Functions
    }
  }
}
```

#### **Validación de Datos**

```javascript
// Validar estructura de datos
allow write: if request.auth != null 
  && request.auth.uid == userId
  && validateUserData(request.resource.data);

function validateUserData(data) {
  return data.keys().hasAll(['name', 'email'])
    && data.name is string
    && data.email is string
    && data.email.matches('.*@.*');
}
```

### 🔐 **Mejores Prácticas**

#### **Configuración Segura**
- **Implementa autenticación robusta** con Firebase Auth
- **Usa reglas granulares** por colección
- **Valida datos** de entrada
- **Nunca almacenes datos sensibles** en documentos públicos
- **Usa Cloud Functions** para operaciones críticas

```javascript
// Ejemplo seguro
exports.updatePoints = functions.https.onCall(async (data, context) => {
  if (!context.auth) throw new functions.https.HttpsError('unauthenticated');
  const userId = context.auth.uid;
  await admin.firestore().collection('ranking').doc(userId).update({
    points: admin.firestore.FieldValue.increment(data.points)
  });
});
```


## Conclusiones

### Lecciones Aprendidas

Este caso demuestra problemas **muy comunes** de Firebase:

- **Facilidad de implementación** vs **complejidad de seguridad**
- **Configuraciones por defecto** demasiado permisivas
- **Falta de conciencia** sobre las implicaciones de seguridad

#### Para Desarrolladores
- **Las reglas de seguridad son tu primera línea de defensa**
- **Prueba las configuraciones** antes de producción
- **Implementa el principio de menor privilegio**



*Este análisis se publica con fines educativos para concientizar sobre la importancia de configuraciones seguras en Firebase y otras plataformas similares.*
