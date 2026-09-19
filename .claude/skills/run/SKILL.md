---
name: run
description: PoE2 Filtre uygulamasını derle ve gerçekten çalıştır (Linux konteynerinde Xvfb altında, ya da Windows'ta bin/poe2filter.exe). Kod değişikliğinden sonra "çalışıyor mu" sorusunu ekran görüntüsüyle yanıtlamak, arayüzü sürmek veya değişiklik sonrası kontrol listesini uygulamak için kullan.
---

# PoE2 Filtre'yi derle ve çalıştır

Bu bir Wails v3 (Go + Svelte 5) tepsi uygulaması. **Her kod değişikliğinden sonra
build alınır ve uygulama açılıp gözle görülür** — test yeşil olması tek başına yeterli
değil.

## Değişiklik sonrası kontrol listesi

```bash
gofmt -l internal/ *.go                 # boş çıktı beklenir
go test ./internal/...                  # build/ios bu ortamda derlenmez, normal
cd frontend && npx svelte-check --tsconfig ./tsconfig.json && npx vite build && cd ..
go build -tags production -o /tmp/poe2filter-gui .
```
Ardından aşağıdaki gibi çalıştır, ekran görüntüsü al ve **görüntüye bak** — siyah kare
"açıldı" demek değil, "açılamadı" demektir. Sonra commit + push.

## Linux konteynerinde çalıştırma (bir kerelik kurulum)

```bash
apt-get update -qq && apt-get install -y -qq \
  libgtk-4-dev libwebkitgtk-6.0-dev libsoup-3.0-dev pkg-config \
  dbus-x11 xvfb x11-apps imagemagick xdotool
```

WebKit'in bubblewrap sandbox'ı konteynerde çalışmaz; sandbox kapatılmazsa uygulama
SIGTRAP ile düşer. Pencere varsayılan olarak tepside gizli açılır, `-show` şart:

```bash
H=/tmp/apphome; mkdir -p $H/.config
(HOME=$H XDG_CONFIG_HOME=$H/.config \
 WEBKIT_FORCE_SANDBOX=0 WEBKIT_DISABLE_SANDBOX_THIS_IS_DANGEROUS=1 \
 WEBKIT_DISABLE_COMPOSITING_MODE=1 WEBKIT_DISABLE_DMABUF_RENDERER=1 GDK_BACKEND=x11 \
 xvfb-run -a -n 81 -s "-screen 0 1280x950x24" dbus-run-session -- \
 /tmp/poe2filter-gui -show > /tmp/gui.log 2>&1 &)
sleep 22
export DISPLAY=:81 XAUTHORITY=$(pgrep -a Xvfb | grep ':81' | sed 's/.*-auth //')
import -window root /tmp/app.png
convert /tmp/app.png -crop 380x645+450+153 +repage /tmp/card.png   # sadece pencere
```

Arayüzü sürmek için `xdotool`: ayarlar dişlisi `mousemove 764 182 click 1`, kaydırma
`click 5` (aşağı) / `click 4` (yukarı), tıklama `mousemove X Y click 1`.
Temizlik: `for pid in $(pgrep -x poe2filter-gui) $(pgrep -x Xvfb); do kill $pid; done`
(`pkill -f poe2filter` kendi kabuğunu da öldürür, kullanma).

Sunucu modu GUI'siz alternatiftir: `go build -tags server,production` ile derlenir,
`localhost:8080`'de aynı arayüzü sunar, Playwright ile sürülebilir.

## Windows'ta (kullanıcının makinesi)

```bash
wails3 build      # bin/poe2filter.exe
```

## Bu ortamın sınırları

- **Ağ:** poe.ninja, api.poe2scout.com ve pathofexile.com egress politikasıyla kapalı;
  uygulama "Filtre güncellenemedi" durumunda açılır, bu beklenen davranış.
  GitHub (NeverSink filtresi dahil) erişilebilir.
- **Filtre çıktısını** gerçek NeverSink dosyasıyla doğrulamak için dosyayı
  `raw.githubusercontent.com/NeverSinkDev/NeverSink-Filter-for-PoE2` üzerinden indirip
  `filter.Inject` ile birleştiren geçici bir Go testi yaz; kural sırası kritik.
- **Bindings:** `go install github.com/wailsapp/wails/v3/cmd/wails3@v3.0.0-beta.23`
  ile CLI kurulur, sonra `wails3 generate bindings -ts -i` (repodaki dosyalar bu
  bayraklarla üretildi; `-i` unutulursa interface'ler class'a dönüşür ve diff patlar).
  CLI olmadan elle yazmak gerekirse çağrı kimliği `FNV-1a 32("main.AppService.<Metot>")`.

## Windows exe'si (release için)

Konteynerden çapraz derlenebilir; ikon ve manifest için önce `.syso` üretilir:

```bash
cd build && wails3 generate syso -arch amd64 -icon windows/icon.ico \
  -manifest windows/wails.exe.manifest -info windows/info.json -out ../wails_windows_amd64.syso
cd .. && (cd frontend && npx vite build)
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build -tags production -trimpath \
  -ldflags "-w -s -H windowsgui -X main.version=<sürüm>" -o poe2filtre-v<sürüm>-windows-amd64.exe .
```

Sürüm iki yerde: `main.go` içindeki `version` ve `build/config.yml`. Release notuna
exe'nin `sha256sum` özetini koy (README böyle söz veriyor). `api.github.com` bu
ortamda kullanıcının kimliğiyle açık, release ve asset yüklemesi REST ile yapılabilir.
