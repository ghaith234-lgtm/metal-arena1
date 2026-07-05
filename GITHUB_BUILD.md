# بناء APK بالسحابة عبر GitHub Actions (بدون تنصيب أي شي)

سيرفرات GitHub المجانية تبنيلك الـ APK. الفكرة: ترفع المشروع لريبو، والـ workflow الموجود بـ `.github/workflows/build-apk.yml` يشتغل تلقائياً ويطلعلك APK جاهز.

## الخطوات

### 1. سوّي ريبو جديد
- ادخل github.com وسجّل دخول
- اضغط **New repository** → سمّيه `metal-arena` → اختار **Private** إذا تريده خاص → **Create repository**

### 2. ارفع ملفات المشروع

**⚠️ أهم نقطة:** ملف `project.godot` لازم يكون **بجذر الريبو مباشرة** — يعني ترفع *محتويات* مجلد metal-arena، مو المجلد نفسه.

**الطريقة أ — من المتصفح:**
بصفحة الريبو الجديد اضغط رابط **uploading an existing file** → اسحب **كل** محتويات مجلد المشروع (بضمنها مجلد `.github` — لا تنساه!) → **Commit changes**

**الطريقة ب — بأوامر git (أضمن):**
```bash
cd metal-arena
git init
git add -A
git commit -m "v1"
git branch -M main
git remote add origin https://github.com/اسم_حسابك/metal-arena.git
git push -u origin main
```

### 3. شغّل البناء
- افتح تبويب **Actions** بالريبو
- إذا انرفع مجلد `.github` صح، البناء يبلش تلقائياً مع أول push
- إذا ما بلش: اختار **Build Android APK** من اليسار → زر **Run workflow**

### 4. نزّل الـ APK
- انتظر ~5-8 دقايق لحد ما تصير علامة ✅ خضراء
- اضغط على الـ run → انزل لقسم **Artifacts** → نزّل **metal-arena-apk**
- ينزل ملف zip → فكه → داخله `metal-arena.apk`
- انقله لموبايلك ونصّبه (فعّل "تثبيت من مصادر غير معروفة")

---

## إذا ما ظهر الـ workflow بتبويب Actions

معناها مجلد `.github` ما انرفع (المتصفح أحياناً يتجاهله). الحل — سوّيه يدوياً:
1. بصفحة الريبو: **Add file → Create new file**
2. باسم الملف اكتب حرفياً: `.github/workflows/build-apk.yml`
3. الصق المحتوى الجاي كامل → **Commit changes**

```yaml
name: Build Android APK

on:
  push:
    branches: [ main ]
  workflow_dispatch:

env:
  GODOT_VERSION: "4.3"

jobs:
  android:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout project
        uses: actions/checkout@v4

      - name: Set up Java 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"

      - name: Download Godot and export templates
        run: |
          wget -q "https://github.com/godotengine/godot/releases/download/${GODOT_VERSION}-stable/Godot_v${GODOT_VERSION}-stable_linux.x86_64.zip"
          wget -q "https://github.com/godotengine/godot/releases/download/${GODOT_VERSION}-stable/Godot_v${GODOT_VERSION}-stable_export_templates.tpz"
          unzip -q "Godot_v${GODOT_VERSION}-stable_linux.x86_64.zip"
          chmod +x "Godot_v${GODOT_VERSION}-stable_linux.x86_64"
          TEMPLATE_DIR="$HOME/.local/share/godot/export_templates/${GODOT_VERSION}.stable"
          mkdir -p "$TEMPLATE_DIR"
          unzip -q "Godot_v${GODOT_VERSION}-stable_export_templates.tpz"
          mv templates/* "$TEMPLATE_DIR/"

      - name: Generate debug keystore
        run: |
          keytool -genkeypair -v -keystore "$HOME/debug.keystore" -alias androiddebugkey -keyalg RSA -keysize 2048 -validity 10000 -storepass android -keypass android -dname "CN=Android Debug,O=Android,C=US"

      - name: Configure Godot editor settings
        run: |
          MINOR="$(echo "$GODOT_VERSION" | cut -d. -f1,2)"
          mkdir -p "$HOME/.config/godot"
          cat > "$HOME/.config/godot/editor_settings-4.tres" <<EOF
          [gd_resource type="EditorSettings" format=3]

          [resource]
          export/android/android_sdk_path = "${ANDROID_HOME}"
          export/android/java_sdk_path = "${JAVA_HOME}"
          export/android/debug_keystore = "${HOME}/debug.keystore"
          export/android/debug_keystore_user = "androiddebugkey"
          export/android/debug_keystore_pass = "android"
          EOF
          cp "$HOME/.config/godot/editor_settings-4.tres" "$HOME/.config/godot/editor_settings-${MINOR}.tres"

      - name: Import project resources
        run: |
          ./Godot_v${GODOT_VERSION}-stable_linux.x86_64 --headless --path . --import || true

      - name: Export debug APK
        run: |
          mkdir -p build
          ./Godot_v${GODOT_VERSION}-stable_linux.x86_64 --headless --path . --export-debug "Android" build/metal-arena.apk
          test -f build/metal-arena.apk
          ls -la build/

      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: metal-arena-apk
          path: build/metal-arena.apk
```

## ملاحظات مهمة

- **تطابق الإصدارات:** البناء مثبّت على Godot **4.3**. إذا بعدين فتحت المشروع محلياً بنسخة أحدث (4.4 / 4.5) وحفظت، غيّر سطر `GODOT_VERSION: "4.3"` بالـ workflow لنفس نسختك.
- كل push جديد للريبو = بناء APK جديد تلقائياً. هيچ صار عندك خط إنتاج: تعدّل الكود → ترفع → تنزّل APK.
- هذا APK بتوقيع debug — كافي تماماً للتجربة والتوزيع بين الأصدقاء. للنشر بالمتاجر لاحقاً نسوي توقيع release.
- صلاحية Internet مفعّلة من هسه بالإعدادات، جاهزة لمرحلة الملتيبلاير LAN.
