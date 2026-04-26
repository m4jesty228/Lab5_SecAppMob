# 🔍 Write-Up — OWASP Uncrackable Level 2

- **Filière** : GCDSTE — Semestre 4  
- **Établissement** : ENSA Marrakech  
- **Auteur** : DOSSAH Landry
- **Application** : UnCrackable-Level2.apk — `owasp.mstg.uncrackable2` — v1.0  

---

## 🎯 Objectif

Ce lab a pour but d'analyser une application Android qui cache sa logique de vérification dans une **bibliothèque native** (fichier `.so`). L'interface est simple : un champ texte + un bouton Verify. Mais la vérification réelle ne se passe pas dans le code Java — elle est déléguée à du code natif compilé, chargé via JNI.

L'objectif final : **retrouver le secret attendu par l'application**.

---

## 🧰 Outils utilisés

| Outil | Rôle |
|---|---|
| `adb` | Installer l'APK sur l'émulateur |
| JADX | Décompiler le code Java de l'APK |
| Ghidra | Analyser la bibliothèque native `.so` |
| Python | Décoder la valeur hexadécimale trouvée |
| `unzip` | Extraire le contenu de l'APK |

---

## 📋 Prérequis

- Bases Android (APK, Activity, Java)
- Savoir lancer une commande dans un terminal
- Savoir installer une application via `adb`
- Avoir déjà vu un peu de Java (classes, méthodes)

---

## 🗺️ Flux logique de l'application

Avant de commencer, voici le chemin complet que suit la vérification :

```
Utilisateur (saisit une chaîne)
        ↓
MainActivity — récupère l'entrée
        ↓
CodeCheck.a(String) — transmet à la couche native
        ↓
CodeCheck.bar(String) — méthode native (JNI)
        ↓
libfoo.so — comparaison avec strncmp
        ↓
true (succès) / false (échec)
```

---

## 📁 Structure du dépôt

```
Lab5_SecAppMob/
├── README.md
├── 01_manifest.png           ← AndroidManifest.xml
├── 02_mainactivity_verify.png ← méthode verify() dans JADX
├── 03_codeccheck_loadlib.png  ← CodeCheck + loadLibrary
├── 04_ghidra_strncmp.png      ← strncmp dans Ghidra
│── 05_app_success.png         ← validation finale "Success!"
└── UnCrackable-Level2.apk
```

---

## 🚀 Partie 1 — Découverte de l'application

### Étape 1 — Installer et lancer l'APK

```cmd
adb install UnCrackable-Level2.apk
```

Lancer ensuite l'application sur l'émulateur. L'interface affiche un champ de saisie et un bouton **VERIFY**.

> 💡 À ce stade, on n'analyse pas encore le code. On observe uniquement le comportement visible.

---

### Étape 2 — Tester des valeurs incorrectes

Saisir quelques chaînes au hasard :

```
test
1234
hello
android
```

**Observation** : l'application affiche systématiquement un message d'erreur — `"Nope..."` / `"That's not it. Try again."` Cela confirme qu'une logique de comparaison existe quelque part dans le code.

---

## 🔍 Partie 2 — Analyse du code Java avec JADX

### Étape 3 — Décompiler l'APK

```cmd
jadx-gui UnCrackable-Level2.apk
```

Ouvrir la classe `MainActivity` dans l'arborescence.

📸 <img width="1316" height="915" alt="image" src="https://github.com/user-attachments/assets/f54930eb-79a2-4af8-8842-04049d67ff46" />


> Le `AndroidManifest.xml` confirme : package `owasp.mstg.uncrackable2`, version 1.0, API min 19, target 28.

---

### Étape 4 — Identifier la méthode de vérification

Dans `MainActivity`, repérer la méthode `verify()` :

```java
public void verify(View view) {
    String string = ((EditText) findViewById(R.id.edit_text)).getText().toString();
    AlertDialog alertDialogCreate = new AlertDialog.Builder(this).create();
    if (this.m.a(string)) {
        alertDialogCreate.setTitle("Success!");
        str = "This is the correct secret.";
    } else {
        alertDialogCreate.setTitle("Nope...");
        str = "That's not it. Try again.";
    }
    alertDialogCreate.setMessage(str);
}
```

📸 <img width="1568" height="556" alt="image" src="https://github.com/user-attachments/assets/8617761a-0f3a-4867-bc92-bbea2852f932" />


**Observation clé** : la chaîne saisie est transmise à `this.m.a(string)`. La vérification n'est pas faite directement ici — elle est déléguée à un autre objet `m` de type `CodeCheck`.

---

## 🔗 Partie 3 — Comprendre CodeCheck et JNI

### Étape 5 — Analyser la classe CodeCheck

Rechercher la classe `CodeCheck` dans JADX :

```java
public class MainActivity extends c {
    private CodeCheck m;

    static {
        System.loadLibrary("foo");
    }
}
```

📸 <img width="1307" height="320" alt="image" src="https://github.com/user-attachments/assets/90a98c3a-0e8b-4b15-8f67-f3a4f619256c" />

**Ce qu'on voit :**

- `System.loadLibrary("foo")` → charge une bibliothèque native nommée `libfoo.so`
- La méthode `bar` est déclarée `native` → son implémentation est en C/C++, pas en Java
- `CodeCheck.a(String)` appelle `bar(String)` qui est la vraie vérification

> 💡 Le mot-clé `native` signifie que le code n'est pas dans les classes Java. Il faut aller chercher dans la bibliothèque compilée.

---

## 📦 Partie 4 — Extraire la bibliothèque native

### Étape 6 — Décompresser l'APK

Un APK est en réalité une archive ZIP. On peut l'extraire :

```cmd
unzip UnCrackable-Level2.apk -d uncrackable_l2
```

Puis lister le dossier `lib` :

```cmd
ls -R uncrackable_l2/lib
```

**Résultat** : plusieurs variantes de `libfoo.so` selon l'architecture (x86, ARM...). Pour l'analyse statique, une seule suffit — prendre `lib/x86/libfoo.so`.

---

## 🔬 Partie 5 — Analyse native avec Ghidra

### Étape 7 — Importer libfoo.so dans Ghidra

Lancer Ghidra, créer un projet, importer `libfoo.so` :

```
File → Import File → uncrackable_l2/lib/x86/libfoo.so
```

Lancer l'analyse automatique (**Analyze → Auto Analyze**).

> 💡 Ghidra est un outil de reverse engineering libre développé par la NSA. Il décompile le code binaire en pseudo-C lisible.

---

### Étape 8 — Trouver la fonction JNI

Les fonctions JNI suivent un schéma de nommage précis basé sur le package, la classe et la méthode Java. Rechercher dans Ghidra :

```
Window → Symbol Table → filtrer : Java_
```

**Fonction trouvée** :

```
Java_sg_vantagepoint_uncrackable2_CodeCheck_bar
```

> C'est la version native de `CodeCheck.bar()` — exactement ce qu'on cherche.

---

### Étape 9 — Lire le pseudo-code et repérer strncmp

Ouvrir la fonction dans le décompilateur Ghidra. On y trouve une comparaison avec `strncmp` :

```c
strcpy((char *)v9, "Thanks for all the fish");
...
if ( (strncmp(v7, (const char *)v9, 0x17u)) == 23
     && !strncmp(v7, (const char *)v9, 0x17u) )
{
    return true;
}
```

📸 <img width="1568" height="777" alt="image" src="https://github.com/user-attachments/assets/82cf8d08-1ab9-4c8f-bf91-16a7d9194f02" />


**Observation** : la chaîne de référence utilisée dans `strncmp` est visible directement dans le pseudo-code Ghidra. Si elle n'était pas en clair, elle serait stockée en hexadécimal ASCII.

---

## 🔓 Partie 6 — Décoder le secret (si stocké en hex)

Dans certaines variantes, la valeur peut apparaître sous forme hexadécimale :

```
6873696620656874206c6c6120726f6620736b6e616854
```

### Étape 10 — Décoder l'hex en ASCII

```python
hex_data = "6873696620656874206c6c6120726f6620736b6e616854"
print(bytes.fromhex(hex_data).decode("ascii"))
```

**Résultat** :

```
hsif eht lla rof sknahT
```

### Étape 11 — Inverser la chaîne

```python
s = "hsif eht lla rof sknahT"
print(s[::-1])
```

**Résultat** :

```
Thanks for all the fish
```

> 💡 Le secret était stocké à l'envers pour ne pas être immédiatement lisible. Une simple inversion en Python suffit à le retrouver.

---

## ✅ Partie 7 — Validation finale

### Étape 12 — Tester le secret dans l'application

Retourner dans l'application sur l'émulateur. Saisir :

```
Thanks for all the fish
```

Cliquer sur **VERIFY**.

📸 <img width="707" height="860" alt="image" src="https://github.com/user-attachments/assets/843df208-84de-4f06-8e6a-7f6c0ee68aca" />


**Résultat** : l'application affiche `"Success! — This is the correct secret."` ✅

---

## 📚 Liens avec OWASP MASTG

| Concept | Référence MASTG |
|---|---|
| Analyse statique APK | MASTG-TEST-0001 |
| Code natif via JNI | MASTG-TEST-0077 |
| Secrets dans les binaires | MASTG-TEST-0006 |
| Reverse engineering `.so` | MASTG-TEST-0079 |

> OWASP MASTG rappelle que les bibliothèques natives sont chargées via JNI et nécessitent un désassembleur (Ghidra, IDA Pro) pour être analysées correctement.

---

## 💡 Ce qu'il faut retenir

Ce lab illustre un principe fondamental en sécurité mobile : **déplacer la logique sensible en code natif ne la protège pas, ça la rend juste plus difficile à lire**.

La bonne méthode d'analyse reste toujours la même :

1. Observer le comportement de l'application
2. Suivre la donnée entrée par l'utilisateur dans le code Java
3. Identifier l'appel vers une bibliothèque native
4. Ouvrir la bibliothèque dans Ghidra
5. Chercher la comparaison de chaînes (`strncmp`)
6. Extraire et décoder la valeur cachée
7. Valider dans l'application

---

## 🛠️ Dépannage

| Problème | Solution |
|---|---|
| JADX ne montre pas `CodeCheck` | Utiliser la recherche globale sur `loadLibrary` |
| Ghidra affiche trop de fonctions | Filtrer avec `Java_` dans la Symbol Table |
| La valeur trouvée est illisible | Vérifier si elle est en hexadécimal ASCII |
| La chaîne décodée est étrange | Essayer de la lire à l'envers |
| L'app refuse encore la réponse | Vérifier majuscules, espaces et orthographe exacte |

---

*Write-up réalisé dans un cadre pédagogique strictement contrôlé — Module M45, ENSA Marrakech, GCDSTE S4.*
