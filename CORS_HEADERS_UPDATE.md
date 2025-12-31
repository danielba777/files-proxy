# CORS Headers Update

## Problem

Browser blockierten den Zugriff auf Bilder von `files.slidescockpit.com` mit:

```
Quellübergreifende (Cross-Origin) Anfrage blockiert: 
Die Gleiche-Quelle-Regel verbietet das Lesen der externen Ressource.
(Grund: CORS-Kopfzeile 'Access-Control-Allow-Origin' fehlt)
```

Dies verhinderte:
- Canvas-Rendering im Browser
- Video-Export mit Bildern
- Thumbnail-Generierung
- Bildverarbeitung im Frontend

## Lösung

CORS-Header in der Nginx-Konfiguration hinzugefügt.

### Änderungen in `nginx.conf`

**Hinzugefügt (Zeilen 16-29):**

```nginx
# CORS-Header für Browser-Zugriff (Canvas, Video-Export, etc.)
add_header Access-Control-Allow-Origin "*" always;
add_header Access-Control-Allow-Methods "GET, HEAD, OPTIONS" always;
add_header Access-Control-Max-Age "86400" always;

# Handle preflight OPTIONS requests
if ($request_method = OPTIONS) {
  add_header Access-Control-Allow-Origin "*";
  add_header Access-Control-Allow-Methods "GET, HEAD, OPTIONS";
  add_header Access-Control-Max-Age "86400";
  add_header Content-Length 0;
  add_header Content-Type text/plain;
  return 204;
}
```

**Wichtig:** 
- `always` sorgt dafür, dass Header auch bei 404/500 Errors gesendet werden
- OPTIONS-Handling für CORS Preflight-Requests
- `Max-Age: 86400` (24h) reduziert Preflight-Requests

## Deployment

### Lokaler Test

```bash
cd /Users/danielbauer/Projekte/SlidesCockpit/files-proxy

# Docker Image bauen
docker build -t files-proxy:latest .

# Lokal testen
docker run -p 8080:80 files-proxy:latest

# CORS-Header testen
curl -I -H "Origin: https://slidescockpit.com" http://localhost:8080/test.webp
```

Erwarte im Output:
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, HEAD, OPTIONS
```

### Production Deployment

**Wenn du Git für Deployment nutzt:**

```bash
cd /Users/danielbauer/Projekte/SlidesCockpit/files-proxy
git add nginx.conf
git commit -m "Add CORS headers for browser access

- Add Access-Control-Allow-Origin: * header
- Handle OPTIONS preflight requests
- Fixes CORS errors when loading images in browser"
git push
```

**Wenn du Coolify/Docker nutzt:**

Das Image wird automatisch neu gebaut wenn du pusht (je nach Coolify-Config).
Oder manuell triggern in Coolify UI.

**Wenn du direkt auf Server deployst:**

```bash
# Auf Server
cd /path/to/files-proxy
git pull
docker-compose down
docker-compose build
docker-compose up -d
```

## Verifizierung nach Deployment

### 1. CORS-Header testen

```bash
curl -I -H "Origin: https://slidescockpit.com" \
  "https://files.slidescockpit.com/imagesets/b-w-portraits/test.webp"
```

Erwarte:
```
HTTP/2 200
access-control-allow-origin: *
access-control-allow-methods: GET, HEAD, OPTIONS
access-control-max-age: 86400
```

### 2. Browser-Test

1. Öffne: https://slidescockpit.com/dashboard/slideshows/cmju9huiy0001l1047ogrvgts
2. DevTools öffnen (F12) → Console
3. **Keine CORS-Fehler** mehr sichtbar
4. **Bilder werden angezeigt**

### 3. OPTIONS Preflight-Test

```bash
curl -X OPTIONS \
  -H "Origin: https://slidescockpit.com" \
  -H "Access-Control-Request-Method: GET" \
  -i "https://files.slidescockpit.com/test.webp"
```

Erwarte:
```
HTTP/2 204
access-control-allow-origin: *
access-control-allow-methods: GET, HEAD, OPTIONS
```

## Sicherheit

**Warum `Access-Control-Allow-Origin: *`?**

- Bilder sind öffentlich zugänglich (kein Auth nötig)
- Keine sensiblen Daten in Bildern
- Ermöglicht Canvas/Video-Rendering von beliebigen Origins
- Standard für Public CDNs (wie Cloudinary, Imgix, etc.)

**Alternative** (wenn du nur SlidesCockpit erlauben willst):

```nginx
add_header Access-Control-Allow-Origin "https://slidescockpit.com" always;
```

Aber `*` ist für öffentliche Bilder-CDNs Standard und empfohlen.

## Auswirkung

**Vorher:**
- ❌ CORS-Fehler im Browser
- ❌ Keine Bilder sichtbar
- ❌ Canvas-Rendering fehlgeschlagen
- ❌ Video-Export nicht möglich

**Nachher:**
- ✅ Bilder laden korrekt
- ✅ Canvas-Rendering funktioniert
- ✅ Video-Export mit Bildern möglich
- ✅ Thumbnail-Generierung funktioniert

## Rollback

Falls Probleme auftreten:

```bash
cd /Users/danielbauer/Projekte/SlidesCockpit/files-proxy
git revert HEAD
git push
```

Oder die CORS-Header-Zeilen aus `nginx.conf` entfernen.

## Monitoring

Nach Deployment überwache:
- Response Header in Browser DevTools
- Keine CORS-Fehler in Browser Console
- Bilder laden korrekt auf slidescockpit.com
- Video-Export funktioniert
