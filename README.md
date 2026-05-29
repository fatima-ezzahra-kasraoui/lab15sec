# Lab — Interception HTTPS & SSL Pinning Bypass sur Android
**Cours :** Sécurité Mobile  
**Outils :** Burp Suite, Frida, ADB, Android Emulator (AVD)  
**Objectif :** Mettre en place un proxy HTTPS, installer un certificat CA, et bypasser la validation SSL via Frida pour intercepter le trafic en clair.

---

## Environnement

| Composant | Valeur |
|---|---|
| OS | Windows 10 |
| Burp Suite | Community Edition v2026.4.3 |
| Frida | 17.9.1 |
| ADB | 1.0.41 |
| Python | 3.13.7 |
| Émulateur | Android Emulator 5554 (API 36 — Pixel 5) |

---<img width="1456" height="823" alt="image" src="https://github.com/user-attachments/assets/7044cdb9-ad7f-44e9-b731-511842361b79" />
<img width="1006" height="817" alt="image" src="https://github.com/user-attachments/assets/9004cc29-91f5-419d-badc-0c88f1d9dc86" />



## Étape 1 — Démarrage de Burp Suite et configuration du listener

### 1.1 Problème : port 8080 occupé

Au démarrage, Burp ne pouvait pas démarrer le proxy sur le port par défaut 8080.

```
Failed to start proxy service on 127.0.0.1:8080.
Check whether another service is already using this port.
```

**Diagnostic :**
```cmd
netstat -ano | findstr :8080
tasklist | findstr 6188
```

Le port était occupé par `AgentService.exe` (PID 6188) — un service système. Il ne faut pas le tuer.

**Solution :** Changer le port de Burp vers **8081**.

<img width="1512" height="791" alt="image" src="https://github.com/user-attachments/assets/1e258cd1-7f94-4112-8707-5135a3ffd6e0" />



---

### 1.2 Listener actif sur le port 8081

Après modification du port, Burp démarre correctement.

<img width="1568" height="678" alt="image" src="https://github.com/user-attachments/assets/2dc6876b-f23f-4763-8298-06ff34ca2d3b" />

---

## Étape 2 — Export du certificat CA Burp

Dans Burp : **Proxy Settings → CA Certificate → Export → Certificate in DER format**  
Fichier sauvegardé : `cacert.der`

<img width="1380" height="868" alt="image" src="https://github.com/user-attachments/assets/38f8c866-f1d8-40a9-8c15-58748f5c7d1f" />


---

## Étape 3 — Installation du certificat CA sur l'émulateur

### 3.1 Push du certificat via ADB

```cmd
set PATH=%PATH%;C:\Users\HP\AppData\Local\Android\Sdk\platform-tools
adb push "C:\Users\HP\Downloads\cacert" /sdcard/cacert.der
```

**Résultat :**
```
C:\Users\HP\Downloads\cacert: 1 file pushed, 0 skipped. 0.4 MB/s (940 bytes in 0.002s)
```

<img width="1327" height="258" alt="image" src="https://github.com/user-attachments/assets/088a570b-7e18-41b1-b017-268f61fc83b4" />

### 3.2 Vérification du fichier sur l'émulateur

```cmd
adb shell ls /sdcard/
```

Le fichier `cacert.der` est présent à la racine de `/sdcard/`.
<img width="365" height="227" alt="image" src="https://github.com/user-attachments/assets/961d17b2-dcde-4454-859f-88995f29e4f4" />

### 3.3 Installation via les paramètres Android

**Settings → Security → Install a certificate → CA certificate**

<img width="375" height="783" alt="image" src="https://github.com/user-attachments/assets/31d5151b-2641-4fbc-b535-e177c2e0f850" />


---

## Étape 4 — Configuration du proxy Wi-Fi sur l'émulateur

**Settings → Network & Internet → Wi-Fi → AndroidWifi → Modify network → Advanced → Proxy Manual**

| Champ | Valeur |
|---|---|
| Proxy hostname | `10.0.2.2` (adresse du PC hôte depuis l'émulateur) |
| Proxy port | `8081` |

<img width="179" height="386" alt="image" src="https://github.com/user-attachments/assets/2fb0af36-74a8-4044-88d8-6d5341cf2bdb" />


---


## Étape 6 — Script SSL Pinning Bypass (Java)

### Script : `sslpin_bypass_universal.js`

Le script couvre :
- **SSLContext.init** — injection d'un TrustManager permissif
- **X509TrustManager** — patch de `checkServerTrusted` / `checkClientTrusted`
- **Conscrypt TrustManagerImpl** — patch Android 7+ (`checkTrusted`, `verifyChain`)
- **OkHttp CertificatePinner** — neutralisation de `check()`
- **WebView** — `onReceivedSslError` → `handler.proceed()`
<img width="1568" height="603" alt="image" src="https://github.com/user-attachments/assets/7a15e94e-3504-4a38-9f9d-8337a67eae33" />



### Exécution sur WebView Shell

```cmd
frida -U -f org.chromium.webview_shell -l "C:\Users\HP\Downloads\sslpin_bypass_universal.js"
```

**Output Frida :**
```
Spawned `org.chromium.webview_shell`. Resuming main thread!
[+] SSL bypass: SSLContext.init patched
[+] SSL bypass: X509TrustManager patches attempted
[+] SSL bypass: com.android.org.conscrypt.TrustManagerImpl patched
[+] Universal SSL pinning bypass installed
[+] SSL bypass: com.android.org.conscrypt.TrustManagerImpl.checkServerTrusted -> allow
[+] SSL bypass: com.android.org.conscrypt.TrustManagerImpl.checkTrusted -> allow
[+] SSL bypass: com.android.org.conscrypt.TrustManagerImpl.verifyChain -> allow
```

<img width="1568" height="557" alt="image" src="https://github.com/user-attachments/assets/bef94103-0447-425c-b1b5-fe7a6a3787e9" />

`

---

## Étape 7 — Validation : interception HTTPS en clair

### 7.1 Navigation HTTPS depuis l'émulateur

Depuis **WebView Shell**, navigation vers :
- `https://example.com`

<img width="472" height="913" alt="image" src="https://github.com/user-attachments/assets/3d9e6573-dad8-4d9e-99c0-c41e4f51a358" />
<img width="1568" height="234" alt="image" src="https://github.com/user-attachments/assets/70b4df3a-49e0-46f2-b752-30c9cbde506c" />
## Étape 8 — Script sslpin_bypass_native.js (Java)
- `https://httpbin.org`
<img width="932" height="670" alt="image" src="https://github.com/user-attachments/assets/3fa89637-60a7-42c9-9667-412421d55c96" />

<img width="1568" height="575" alt="image" src="https://github.com/user-attachments/assets/04e9cec8-1391-4121-9ac2-dd314c9f6f03" />

<img width="424" height="884" alt="image" src="https://github.com/user-attachments/assets/1c81a851-1b9a-4ad7-a367-a32c96d50c00" />
<img width="1488" height="401" alt="image" src="https://github.com/user-attachments/assets/25f45c07-1ce4-4398-b573-e41133e2d77f" />



### etape 9 : Détail d'une requête interceptée

<img width="1440" height="453" alt="image" src="https://github.com/user-attachments/assets/d5cf82f4-88ac-4c55-bcae-35d80469e914" />

> `Screenshot_2026-05-29_050004.png` / `Screenshot_2026-05-29_050017.png`

---

## Résultats

| Étape | Statut |
|---|---|
| Burp proxy actif sur port 8081 | ✅ |
| Certificat CA exporté et installé sur l'émulateur | ✅ |
| Proxy Wi-Fi configuré (10.0.2.2:8081) | ✅ |
| Frida server lancé sur l'émulateur | ✅ |
| Script SSL bypass chargé et actif | ✅ |
| Trafic HTTPS intercepté en clair dans Burp | ✅ |



## Commandes de référence

```cmd
# Vérifier le port occupé
netstat -ano | findstr :8080
tasklist | findstr <PID>

# Push certificat
adb push "C:\Users\HP\Downloads\cacert" /sdcard/cacert.der

# Vérifier fichier sur émulateur
adb shell ls /sdcard/

# Lancer frida-server
adb shell "/data/local/tmp/frida-server -l 0.0.0.0"

# Lister les apps
frida-ps -Uai

# Lancer bypass SSL
frida -U -f org.chromium.webview_shell -l "C:\Users\HP\Downloads\sslpin_bypass_universal.js"
```
