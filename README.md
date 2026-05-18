# Lab 5 — OWASP UnCrackable Niveau 2

**Module :**  Securite des apps mobile
**Niveau :** 4IIR — Semestre 2  
**Application analysée :** OWASP UnCrackable Level 2  

---

## 1. Introduction

Ce laboratoire a pour objectif d'analyser une application Android intentionnellement vulnérable — OWASP UnCrackable Level 2 — afin de retrouver une chaîne secrète cachée dans du code natif. L'application intègre plusieurs mécanismes de protection (détection de root, détection de débogueur, code natif) que l'ingénierie inverse permet de contourner et d'analyser.

---

## 2. Outils Utilisés

| Outil | Version | Rôle |
|-------|---------|------|
| JADX-GUI | 1.5.5 | Décompilation du bytecode Java |
| Ghidra | 12.0.4 | Analyse du code natif `.so` |
| ADB | - | Installation de l'APK sur l'émulateur |
| PowerShell | - | Extraction du contenu de l'APK |

---

## 3. Analyse Statique Java avec JADX

### 3.1 Installation de l'APK
```bash
adb install UnCrackable-Level2.apk
<img width="299" height="57" alt="installaton apk " src="https://github.com/user-attachments/assets/82c81d7f-567d-49c2-ad26-5e047e8780be" />

L'application se ferme immédiatement sur un émulateur rooté. En analysant `MainActivity` avec JADX, on identifie les vérifications effectuées au démarrage :

```java
if (b.a() || b.b() || b.c()) {
    a("Root detected!");
}
if (a.a(getApplicationContext())) {
    a("App is debuggable!");
}
```

La méthode `a()` affiche un dialog puis appelle `System.exit(0)`.

### 3.2 Identification du mécanisme de vérification

La méthode `verify()` dans `MainActivity` transmet la saisie utilisateur à la classe `CodeCheck` :

```java
public void verify(View view) {
    String string = ((EditText) findViewById(R.id.edit_text)).getText().toString();
    if (this.m.a(string)) {
        // Succès
    }
}
```

### 3.3 Découverte du code natif

L'analyse de la classe `CodeCheck` révèle que la vérification est déléguée à une méthode native :

<img width="435" height="161" alt="codecheck" src="https://github.com/user-attachments/assets/0baedc92-88bd-4aff-8243-04a92683239e" />

La logique réelle est donc compilée dans une bibliothèque native `libfoo.so`.

## 4. Extraction de l'APK

Un fichier APK est une archive ZIP. On l'extrait pour accéder à la bibliothèque native :

<img width="970" height="91" alt="extraction" src="https://github.com/user-attachments/assets/ee2b3c5f-19c1-4329-82fa-140e3fa517fa" />

Structure obtenue :

<img width="789" height="389" alt="contenu extraction" src="https://github.com/user-attachments/assets/4654081c-0be8-4321-831f-ba76bbecfacd" />

---

## 5. Analyse Native avec Ghidra

### 5.1 Import et configuration

1. Créer un nouveau projet Ghidra (`lab5`)
2. **File → Import File** → sélectionner `lib/x86/libfoo.so`
3. Ouvrir le CodeBrowser → lancer l'analyse automatique
   
<img width="1391" height="349" alt="ghidra opening" src="https://github.com/user-attachments/assets/e8e67f14-a147-4c4b-aeac-332f9c8c4f14" />

Ghidra détecte : architecture `x86`, `Little Endian`, `32-bit`, compilé avec `gcc`.

### 5.2 Localisation de la fonction cible

Dans le **Symbol Tree → Functions**, on identifie la fonction native correspondant à `CodeCheck.bar()` :

```
Java_sg_vantagepoint_uncrackable2_CodeCheck_bar
```

### 5.3 Vue désassemblée

La chaîne secrète est construite sur la pile via des instructions MOV avec des valeurs hexadécimales en little endian :

<img width="844" height="356" alt="hex code" src="https://github.com/user-attachments/assets/a1607843-f86b-45ef-9dbf-7f29dde2e8ff" />


### 5.4 Vue décompilée

Le décompilateur Ghidra (**Window → Decompiler**) produit un pseudo-code C lisible :

<img width="844" height="356" alt="hex code" src="https://github.com/user-attachments/assets/b8c85ae0-1832-4b00-9a41-0377048ac467" />

---

## 6. Décodage des Valeurs Hexadécimales (Python)

Pour confirmer le décodage manuel des instructions MOV :

<img width="1153" height="164" alt="image" src="https://github.com/user-attachments/assets/57375864-5746-4847-a36e-d149cb89263f" />

**Résultat :**
```
Chaîne secrète: Thanks for all the fish
```

---

## 7. Résultats

| Propriété | Valeur |
|-----------|--------|
| Chaîne secrète | `Thanks for all the fish` |
| Longueur | 23 caractères (`0x17`) |
| Emplacement | `libfoo.so` |
| Fonction native | `Java_sg_vantagepoint_uncrackable2_CodeCheck_bar` |
| Méthode de comparaison | `strncmp(__s1, local_30, 0x17)` |

---

## 8. Conclusion

Ce laboratoire a permis de mettre en pratique les techniques de rétro-ingénierie sur une application Android contenant du code natif. L'analyse statique avec JADX a permis d'identifier le flux de vérification jusqu'à la méthode native. L'analyse de `libfoo.so` avec Ghidra, notamment via la vue décompilée, a permis de retrouver directement la chaîne secrète `"Thanks for all the fish"` sans exécuter l'application.

Ce cas illustre que même le code natif ne constitue pas une protection suffisante contre l'analyse statique lorsque les chaînes de caractères sont stockées en clair en mémoire.
