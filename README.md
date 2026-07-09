# Rikipo Trader — build APK przez GitHub Actions

Ten projekt buduje aplikację **Rikipo Trader** jako APK w chmurze GitHuba.
Po wgraniu tu masz pełny, powtarzalny build — a co najważniejsze:
**aktualizacje instalują się „na wierzch", bez odinstalowywania starej wersji.**

---

## Dlaczego wcześniej musiałeś odinstalowywać za każdym razem

Android odmawia aktualizacji APK w dwóch sytuacjach:

1. **Ten sam `versionCode`** — system widzi „tę samą wersję" i nie pozwala nadpisać.
2. **Inny podpis (keystore)** — to najczęstsza przyczyna. Jeśli build generuje
   za każdym razem NOWY losowy debug-keystore, każdy APK jest podpisany innym
   kluczem. Android traktuje inny podpis jak obcą aplikację i **blokuje**
   instalację na wierzch („pakiet w konflikcie z istniejącym"). Jedyne wyjście
   to odinstalować.

Ten projekt rozwiązuje oba:
- `versionCode` = numer builda GitHuba → **zawsze rośnie**.
- Podpis stałym keystore trzymanym w sekrecie → **zawsze ten sam klucz**.

---

## Co jest w pliku (edytujesz), a co się generuje samo

| Plik / folder | Co to jest | Czy edytujesz |
|---|---|---|
| `www/index.html` | **CAŁA TWOJA APLIKACJA.** Tu robisz wszystkie zmiany. | ✅ TAK — to jedyny plik z kodem aplikacji |
| `capacitor.config.json` | ID i nazwa apki, `webDir` | rzadko |
| `package.json` | wersje Capacitor | rzadko |
| `.github/workflows/build.yml` | przepis na build (wersjonowanie, podpis) | rzadko |
| `KEYSTORE_B64.txt` | keystore w Base64 — do wklejenia jako SEKRET | jednorazowo |
| `android/` | projekt natywny Android | ❌ NIE — generuje się sam przy buildzie |
| `node_modules/` | zależności | ❌ NIE — instalują się same |

**W praktyce: chcesz zmienić coś w aplikacji → edytujesz WYŁĄCZNIE `www/index.html`.**
Reszta jest już ustawiona.

---

## Konfiguracja jednorazowa (robisz raz)

### 1. Wgraj pliki do repozytorium GitHub
Utwórz repo (np. `rikipo-trader`) i wgraj do niego całą zawartość tego folderu
**oprócz** `KEYSTORE_B64.txt` (jego użyjesz w kroku 3, nie commituj go).

### 2. Ustaw sekrety (Settings → Secrets and variables → Actions → New repository secret)

Dodaj cztery sekrety:

| Nazwa sekretu | Wartość |
|---|---|
| `KEYSTORE_B64` | cała zawartość pliku `KEYSTORE_B64.txt` (jedna długa linia Base64) |
| `KEYSTORE_PASS` | `rikipo123` |
| `KEY_ALIAS` | `rikipo` |
| `KEY_PASS` | `rikipo123` |

> Hasła `rikipo123` pasują do dołączonego keystore. Jeśli chcesz własne —
> wygeneruj nowy keystore (patrz sekcja na dole) i zmień hasła tutaj.

### 3. Uruchom pierwszy build
Wejdź w zakładkę **Actions** → **Build Rikipo Trader APK** → **Run workflow**
(albo po prostu zrób dowolny commit do gałęzi `main`).

Po ~3–6 min build się skończy. APK znajdziesz w dwóch miejscach:
- **Releases** (prawa kolumna repo) — plik `rikipo-trader-v1.0.X.apk`
- **Actions → dany build → Artifacts** — `rikipo-trader-apk`

Pobierz APK na telefon i zainstaluj. Za pierwszym razem Android poprosi o zgodę
na „instalację z nieznanych źródeł" — zezwól.

---

## Jak aktualizować aplikację (to robisz za każdym razem)

1. Otwórz `www/index.html` w repo (ikona ołówka „Edit" na GitHubie wystarczy
   na drobne zmiany, albo podmień cały plik).
2. Wklej nową wersję / nanieś zmiany → **Commit changes**.
3. Commit automatycznie uruchamia build. Poczekaj ~5 min.
4. Wejdź w **Releases**, pobierz najnowszy APK.
5. Zainstaluj **na wierzch — BEZ odinstalowywania starej wersji.**
   Zadziała, bo podpis jest ten sam, a `versionCode` wyższy.

Twoje dane (dziennik, ustawienia, watchlista w localStorage) **zostają**,
bo to aktualizacja tej samej aplikacji, a nie nowa instalacja.

---

## Gdybyś chciał własny keystore (opcjonalne)

Dołączony keystore jest w porządku do użytku prywatnego. Jeśli jednak chcesz swój:

```bash
keytool -genkeypair -v \
  -keystore rikipo-release.keystore \
  -alias rikipo -keyalg RSA -keysize 2048 -validity 10000 \
  -storepass TWOJE_HASLO -keypass TWOJE_HASLO \
  -dname "CN=Rikipo Trader, O=Rikipo, C=DE"

base64 -w0 rikipo-release.keystore
```

Wynik `base64` wklej do sekretu `KEYSTORE_B64`, a hasła do `KEYSTORE_PASS` / `KEY_PASS`.

> ⚠️ WAŻNE: **nie zmieniaj keystore po pierwszej instalacji.** Zmiana klucza
> = zmiana podpisu = znów trzeba by odinstalowywać. Trzymaj jeden na zawsze.

---

## Najczęstsze problemy

- **„Aplikacja nie została zainstalowana"** przy aktualizacji → prawie zawsze
  konflikt podpisu. Upewnij się, że sekret `KEYSTORE_B64` się nie zmienił między
  buildami. Jeśli poprzednią wersję instalowałeś ze starego, inaczej podpisanego
  APK — ten jeden raz odinstaluj, potem już będzie gładko.
- **Build czerwony na kroku „versionCode"** → sprawdź, czy `android/app/build.gradle`
  ma linie `versionCode` i `versionName` (Capacitor generuje je domyślnie).
- **Dane zniknęły po aktualizacji** → to znaczy, że jednak była to nowa instalacja
  (inny podpis). Patrz punkt pierwszy.
