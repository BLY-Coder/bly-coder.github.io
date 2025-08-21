---
layout: single
title: "Firebase Exposed: When Insecure Configurations Compromise Educational Platforms"
excerpt: Analysis of critical vulnerabilities in Firebase configurations that allow user data manipulation, personal information exposure, and unauthorized access to scoring systems.
date: 2025-08-21
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

## Introduction: The Dark Side of Firebase

**Hello everyone!** Today we're going to analyze a very common but dangerous problem in web development: **insecure Firebase configurations**. During a security investigation, I discovered multiple vulnerabilities in an educational platform that uses Firebase as backend, exposing user data and allowing unauthorized system manipulation.

![Firebase Security](/assets/images/firebase-exposed/logo.jpeg)

### What is Firebase and Why is it Popular?

Firebase is Google's app development platform that offers a **NoSQL database (Firestore)**, **authentication**, **hosting**, and many other services. Its popularity is due to ease of implementation, but this simplicity often leads to poor security configurations.

## The Case: Compromised Educational Platform

### Discovery Context

The analyzed platform is an educational site that includes:
- **Online course system**
- **User community** with point system
- **Rankings** based on activity
- **User profiles** with personal data

### Vulnerability 1: Firebase Credentials Exposure

#### **The Problem**
Complete Firebase credentials are **exposed in client-side JavaScript code**, accessible from any browser.

#### **Compromised File**
```
GET /assets/index-B1xorCBP.js HTTP/2
```

#### **Exposed Credentials**
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

#### **Why is this Problematic?**
While Google says these credentials can be public, **the real problem is misconfigured security rules** that allow unauthorized access.

![Firebase Config Exposed](/assets/images/firebase-exposed/fire1.png)

### Vulnerability 2: Insecure Firestore Rules

#### **Direct Access to Collections**
The `ranking` collection has permissions that allow **unauthorized read and write access**:

```javascript
// Unauthorized read
Getting [DOC_ID] from ranking

// Unauthorized write  
Updated fields(Doc ID: [DOC_ID])
{"points":4510,"number":"XXXXX","name":"XXXX"}
```

#### **Real Impact**
Users can:
- **Read data** from other users
- **Modify their own points** arbitrarily
- **Manipulate rankings** without restrictions
- **Access personal information** of other users

### Vulnerability 3: Personal Data Exposure

#### **Compromised Information**
Documents contain sensitive data partially exposed:

```json
{
    "points": 4510,
    "number": "XXXXX",  // Part of phone number
    "name": "XXXX"
}
```

#### **Exposed Data**
- **Parts of user phone numbers**
- **Real usernames**
- **Community scores and activity**
- **Document IDs** that may reveal patterns



## Practical Demonstration

### Access from Firebase Console

With the exposed credentials, anyone can:

1. **Access Firebase Console** with the Project ID
2. **Connect to Firestore** directly
3. **Read collections** without authentication
4. **Modify documents** if permissions allow

### Point Manipulation

```javascript
// From browser console
const db = firebase.firestore();

// Read other users' data
db.collection('ranking').get().then((snapshot) => {
    snapshot.forEach((doc) => {
        console.log(doc.id, doc.data());
    });
});

// Modify my own points
db.collection('ranking').doc('[MY_DOC_ID]').update({
    points: 999999
});
```

![Data Exposure](/assets/images/firebase-exposed/fire2.png)

## Risks and Impact

### 🎯 **For Users**

#### **Compromised Privacy**
- **Exposure of partial phone numbers**
- **Real names** publicly visible
- **Activity and progress** accessible without authorization

#### **System Manipulation**
- **False rankings** affecting credibility
- **Unfair competition** in the point system
- **Loss of motivation** due to manipulated rankings

### 🏢 **For the Platform**

#### **Loss of Integrity**
- **Gamification system** completely compromised
- **User trust** damaged
- **False metrics** for business analysis

#### **Legal Risks**
- **GDPR non-compliance** due to data exposure
- **Legal liability** for information leakage
- **Potential sanctions** from data protection authorities

## Common Insecure Configurations

```javascript
// NEVER do this
allow read, write: if true;  // DANGEROUS!

// Weak rules
allow read, write: if request.auth != null;  // Insufficient
```

## Security Recommendations

### 🛡️ **Secure Firestore Configuration**

#### **Robust Security Rules**

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Only owner can read/write their data
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
    
    // Rankings read-only for authenticated users
    match /ranking/{docId} {
      allow read: if request.auth != null;
      allow write: if false;  // Only through Cloud Functions
    }
  }
}
```

#### **Data Validation**

```javascript
// Validate data structure
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

### 🔐 **Best Practices**

#### **Secure Configuration**
- **Implement robust authentication** with Firebase Auth
- **Use granular rules** per collection
- **Validate input data**
- **Never store sensitive data** in public documents
- **Use Cloud Functions** for critical operations

```javascript
// Secure example
exports.updatePoints = functions.https.onCall(async (data, context) => {
  if (!context.auth) throw new functions.https.HttpsError('unauthenticated');
  const userId = context.auth.uid;
  await admin.firestore().collection('ranking').doc(userId).update({
    points: admin.firestore.FieldValue.increment(data.points)
  });
});
```



## Conclusions

### Lessons Learned

This case demonstrates **very common** Firebase problems:

- **Ease of implementation** vs **security complexity**
- **Default configurations** too permissive
- **Lack of awareness** about security implications

#### For Developers
- **Security rules are your first line of defense**
- **Test configurations** before production
- **Implement the principle of least privilege**



*This analysis is published for educational purposes to raise awareness about the importance of secure configurations in Firebase and similar platforms.*
