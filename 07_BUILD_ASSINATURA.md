# 07 — Build, APK e Assinatura

## Fases de distribuição

| Fase | Formato | Uso | Assinatura |
|---|---|---|---|
| Desenvolvimento | APK debug | Você, emulador | Automática (debug keystore) |
| Alpha com amigos | APK release assinado | 3-5 pessoas | Keystore próprio |
| Play Store | AAB release assinado | Público | Play App Signing |

## Passo 1 — Gerar keystore de release

```bash
keytool -genkeypair -v \
  -keystore gastos-release.jks \
  -alias gastos \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -storepass SUA_SENHA_FORTE \
  -keypass SUA_SENHA_FORTE \
  -dname "CN=Seu Nome, OU=Mobile, O=SeuApp, L=Sao Paulo, ST=SP, C=BR"
```

- Guarde `gastos-release.jks` em local seguro e backup (se perder, não pode atualizar o app na Play Store).
- **Nunca** commite o `.jks` nem senhas no git.

## Passo 2 — `keystore.properties` (fora do git)

Na raiz do projeto:
```properties
storeFile=gastos-release.jks
storePassword=SUA_SENHA_FORTE
keyAlias=gastos
keyPassword=SUA_SENHA_FORTE
```
Adicione `keystore.properties` ao `.gitignore`.

## Passo 3 — Configurar assinatura em `app/build.gradle.kts`

```kotlin
import java.util.Properties

val keystoreProps = Properties().apply {
    val f = rootProject.file("keystore.properties")
    if (f.exists()) load(f.inputStream())
}

android {
    signingConfigs {
        create("release") {
            if (keystoreProps.isNotEmpty()) {
                storeFile = rootProject.file(keystoreProps["storeFile"] as String)
                storePassword = keystoreProps["storePassword"] as String
                keyAlias = keystoreProps["keyAlias"] as String
                keyPassword = keystoreProps["keyPassword"] as String
            }
        }
    }
    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
    }
}
```

## Passo 4 — Gerar APK de release (para amigos)

```bash
./gradlew assembleRelease
# saída: app/build/outputs/apk/release/app-release.apk
```

Distribuição: Google Drive, WhatsApp, ou Firebase App Distribution (recomendado
para receber feedback + crash logs).

## Passo 5 — Gerar AAB (para Play Store)

```bash
./gradlew bundleRelease
# saída: app/build/outputs/bundle/release/app-release.aab
```

O AAB é **só para upload no Play Console**. Não é instalável diretamente.

## Passo 6 — Play App Signing
Ao criar o app no Play Console, o Google gera um keystore de assinatura dele.
Você envia o AAB assinado com sua keystore de *upload*. A keystore de upload
pode ser perdida e recuperada via Google; a do Play App Signing é gerenciada
pelo Google. Ainda assim, **faça backup** da sua.

## Checklist release
- [ ] `versionCode` incrementado
- [ ] `versionName` atualizado
- [ ] `isMinifyEnabled = true` (testar se Room/Compose continuam OK)
- [ ] Testado em device físico (não só emulador)
- [ ] `keystore.properties` no `.gitignore`
- [ ] APK release testado antes de enviar AAB
