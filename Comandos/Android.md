Para listar dispositivos virtuales android

Windows / linux / macos

```
emulator -list-avds
```

Para ejecutar un dispositivo

Windows / linux / macos

```
emulator -avd Pixel_6_API_33
emulator -avd Moto_G7_Nestor -no-snapshot-load

```

Lista los emuladores en ejecución

```
adb devices

```

Reiniciar el emulador específico

```
adb -s <emulator-id> reboot
```

O para cerrar todos los emuladores

```
adb kill-server
adb start-server

```

Hacer bundle apk con gradle

```
sudo ./gradlew assembleRelease
```

## 🔹 Gestión del emulador/dispositivo

```bash
adb devices                  # Lista los dispositivos y emuladores conectados
adb -s emulator-5554 shell   # Abrir una shell dentro del emulador específico

```

## 🔹 Copiar archivos

```bash
adb pull <ruta-emulador> <ruta-pc>   # Copiar archivo del emulador a la PC
adb push <ruta-pc> <ruta-emulador>   # Copiar archivo de la PC al emulador

```

**Ejemplos:**

```bash
adb pull /sdcard/Download/archivo.txt ./archivo.txt
adb pull /data/data/com.tuapp/databases/mi_db.db ./mi_db.db
adb push ./archivo.txt /sdcard/Download/

```

## 🔹 Limpiar datos (_wipe data_)

### Solo tu app:

```bash
adb shell pm clear com.tuapp   # Borra datos y cache de la app (como "Clear Storage")

```

### Todo el emulador (factory reset):

```bash
emulator -avd <nombre_AVD> -wipe-data   # Reinicia el emulador como recién creado

```

## 🔹 Navegación interna

```bash
adb shell       # Entra a la terminal del emulador
ls              # Listar archivos
cd <ruta>       # Moverse entre carpetas
exit            # Salir de la shell

```

## 🔹Este es el Comando que mejor me funciono para hacer una apk con expo

```jsx
npx eas-cli build --platform android --profile preview --local --clear-cache
```

## 🔹Version de comando mejorada con nombre

```jsx
npx eas-cli build --platform android --profile preview --local --clear-cache --output nombre_apk.apk
```