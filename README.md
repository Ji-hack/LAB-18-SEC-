# LAB18 – Analyse Dynamique et Extraction d'Identifiants Firebase dans FireStorm

**Réalisé par : Jihad**
**Module : Sécurité Mobile**
**Challenge : FireStorm (PwnSec)**
**Catégorie : Android Reverse Engineering**

---

# Introduction

Ce laboratoire porte sur l'analyse d'une application Android nommée *FireStorm*. L'objectif est d'identifier un mécanisme caché de génération de mot de passe, d'exécuter cette fonctionnalité à l'aide d'un framework d'instrumentation dynamique puis d'utiliser les informations obtenues pour accéder à une base de données Firebase contenant le flag du challenge.

Le scénario combine plusieurs domaines :

* Reverse engineering Android
* Analyse statique Java
* Instrumentation dynamique avec Frida
* Interaction avec Firebase Authentication
* Extraction de données depuis Firebase Realtime Database

---

# Environnement de test

## Plateforme

* Windows 10
* Android Studio Emulator
* Android 10 (API 29)
* Architecture x86_64

## Outils utilisés

| Outil       | Fonction                       |
| ----------- | ------------------------------ |
| JADX-GUI    | Décompilation Java             |
| Frida       | Instrumentation dynamique      |
| frida-tools | Exécution des scripts Frida    |
| Python 3    | Développement des scripts      |
| Pyrebase4   | Communication avec Firebase    |
| ADB         | Gestion de l'émulateur Android |

---

# Déploiement de l'application

L'APK est installé sur l'émulateur Android :

```bash id="x9j4mc"
adb install FireStorm.apk
```

Vérification du bon fonctionnement de Frida :

```bash id="r8q37z"
frida --version

frida-ps -U
```

Le serveur Frida est ensuite transféré puis exécuté sur l'émulateur.

```bash id="4rj6t0"
adb push frida-server /data/local/tmp/

adb shell chmod +x /data/local/tmp/frida-server

adb shell /data/local/tmp/frida-server
```

Cette étape permet l'injection future de scripts JavaScript dans l'application.

---

# Analyse statique de l'APK

L'application est ouverte dans JADX afin d'étudier son comportement interne.

Package principal identifié :

```text id="0i7qya"
com.pwnsec.firestorm
```

L'analyse de MainActivity révèle une méthode particulièrement intéressante.

---

# Étude de la méthode Password()

Une fonction nommée `Password()` est présente dans le code source.

Cette méthode :

1. Lit plusieurs chaînes stockées dans les ressources Android.
2. Concatène certaines portions de ces chaînes.
3. Génère une nouvelle valeur.
4. Transmet cette valeur à une fonction native.

Architecture observée :

```text id="sqj1kp"
strings.xml
      ↓
Concaténation
      ↓
Password()
      ↓
generateRandomString()
      ↓
Mot de passe Firebase
```

Un élément important ressort immédiatement :

La méthode n'est appelée nulle part dans le cycle normal d'exécution de l'application.

Aucun bouton, listener ou callback ne déclenche cette fonctionnalité.

---

# Informations Firebase découvertes

L'analyse du fichier :

```text id="cl4hh6"
res/values/strings.xml
```

permet de récupérer plusieurs paramètres Firebase.

Éléments identifiés :

* Clé API Firebase
* Adresse de la base de données
* Adresse email utilisée pour l'authentification

Ces informations indiquent que l'application s'appuie sur Firebase Authentication et Firebase Realtime Database.

---

# Analyse du mécanisme de génération

Le mot de passe n'est pas stocké directement dans l'application.

Le processus repose sur deux couches :

### Couche Java

Construction d'une chaîne intermédiaire à partir des ressources Android.

### Couche Native

Appel de :

```java id="vj4jzw"
generateRandomString()
```

Cette fonction est implémentée dans une bibliothèque native.

Le résultat retourné constitue le mot de passe Firebase attendu par le serveur.

---

# Instrumentation dynamique avec Frida

Puisque la méthode Password() n'est jamais exécutée naturellement, il est nécessaire de provoquer son exécution.

Un script Frida est développé pour rechercher les instances actives de MainActivity dans la machine virtuelle Android.

Principe :

```text id="7okx5s"
Recherche MainActivity
         ↓
Récupération de l'instance
         ↓
Appel direct de Password()
         ↓
Exécution de generateRandomString()
         ↓
Affichage du mot de passe généré
```

Le script est injecté lors du démarrage de l'application.

```bash id="y18t8m"
frida -U -f com.pwnsec.firestorm -l firestorm.js
```

---

# Résultat obtenu

Après quelques secondes, Frida identifie l'activité principale et exécute la méthode cachée.

Exemple de sortie :

```text id="d1z7dy"
[+] Firebase Password Retrieved
```

Le mot de passe généré est alors affiché dans la console Frida.

Ce mot de passe est indispensable pour la phase suivante.

Il doit être utilisé immédiatement puisque sa génération dépend d'un algorithme dynamique.

---

# Connexion au service Firebase

Une fois les identifiants récupérés, un script Python est utilisé pour s'authentifier.

Architecture :

```text id="n7lvji"
Email Firebase
        +
Mot de passe récupéré via Frida
        ↓
Firebase Authentication
        ↓
Obtention du Token JWT
        ↓
Accès à la base Realtime Database
```

La bibliothèque Pyrebase simplifie les interactions avec les services Firebase.

---

# Extraction du contenu de la base

Après authentification, le script interroge directement la base de données.

Flux complet :

```text id="vzw2h8"
APK
 ↓
Analyse JADX
 ↓
Découverte de Password()
 ↓
Frida
 ↓
Mot de passe Firebase
 ↓
Authentification
 ↓
Realtime Database
 ↓
Flag
```

L'accès est accepté par Firebase et les données sont retournées avec succès.

---

# Résultat final

La base contient le flag du challenge :

```text id="kgkmgb"
PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!_0R_!5_!t???}
```

---

# Analyse de sécurité

Cette application présente plusieurs faiblesses de conception.

## Fonctionnalité sensible exposée

La méthode générant le mot de passe est intégrée au client Android.

Même si elle n'est jamais appelée par l'interface graphique, elle reste accessible à un analyste.

---

## Secrets présents côté client

Les paramètres Firebase sont stockés directement dans les ressources de l'application.

Un attaquant peut facilement les récupérer via décompilation.

---

## Protection insuffisante

Aucun mécanisme n'empêche :

* L'instrumentation Frida
* L'appel direct des méthodes Java
* L'exécution de fonctions natives

---

# Conclusion

Ce laboratoire démontre l'efficacité de l'analyse combinée statique et dynamique dans le contexte Android.

L'utilisation de JADX permet d'identifier une fonctionnalité cachée tandis que Frida permet son exécution sans modifier l'application. Les informations obtenues servent ensuite à contourner le mécanisme d'authentification Firebase et à accéder aux données protégées.

Cette étude illustre également un principe fondamental de la sécurité mobile : toute donnée ou logique présente côté client doit être considérée comme accessible à un attaquant disposant des outils adéquats.
