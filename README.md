# Mini-Project-2-LLM-powered-RESTful-Web-Service-
# Darija Translator — LLM-powered RESTful Web Service

Translate text from English, French, Spanish, or German into **Moroccan Arabic Dialect (Darija)** using Google Gemini 1.5 Flash, served via a Jakarta EE REST endpoint with Basic Authentication.

---

## Project structure

```
darija-translator/          ← Maven / Jakarta EE REST service (WAR)
php-client/                 ← PHP web client
python-client/              ← Python CLI client
react-native-client/        ← React Native mobile app
darija-translator-extension/ ← Chrome Extension (Manifest V3)
```

---

## 1. Server setup

### Prerequisites
- JDK 17+
- Maven 3.9+
- GlassFish 7 or Payara 6 (Jakarta EE 10)
- A Google Gemini API key (free tier: https://ai.google.dev/pricing#1_5flash)

### Build
```bash
cd darija-translator
mvn clean package
```

### Configure Gemini API key
```bash
export GEMINI_API_KEY=your_key_here
```

### Create a user in GlassFish
```bash
# Start GlassFish
./glassfish7/bin/asadmin start-domain

# Create user "apiuser" in group "translator-group"
./glassfish7/bin/asadmin create-file-user \
  --groups translator-group \
  --authrealmname file \
  apiuser
# Enter password when prompted
```

### Deploy
```bash
./glassfish7/bin/asadmin deploy darija-translator/target/darija-translator.war
```

The service is now at: `http://localhost:8080/darija-translator/api/translate`

---

## 2. REST API reference

### Health check (no auth)
```bash
curl http://localhost:8080/darija-translator/api/translate/health
```

### POST — translate JSON body
```bash
curl -X POST http://localhost:8080/darija-translator/api/translate \
  -H "Content-Type: application/json" \
  -u apiuser:secret \
  -d '{"text": "Hello, how are you?", "sourceLang": "English"}'
```

**Response:**
```json
{
  "original":    "Hello, how are you?",
  "sourceLang":  "English",
  "targetLang":  "Moroccan Arabic (Darija)",
  "translation": "أهلاً، كيف داير؟"
}
```

### GET — quick translate
```bash
curl -u apiuser:secret \
  "http://localhost:8080/darija-translator/api/translate?text=Thank+you&sourceLang=English"
```

### Test wrong credentials (expect HTTP 401)
```bash
curl -v -u wrong:credentials \
  -X POST http://localhost:8080/darija-translator/api/translate \
  -H "Content-Type: application/json" \
  -d '{"text": "test"}'
```

---

## 3. Chrome Extension

1. Open Chrome → `chrome://extensions/`
2. Enable **Developer mode** (top right)
3. Click **Load unpacked** → select the `darija-translator-extension/` folder
4. Click the extension icon to open the side panel
5. Right-click any selected text → **Translate to Darija**
6. Set your server URL and credentials in the Settings panel

---

## 4. PHP client

```bash
cd php-client
php -S localhost:9090
# Open http://localhost:9090
```

Edit `API_USER` / `API_PASS` / `API_URL` constants at the top of `index.php`.

---

## 5. Python client

```bash
pip install requests
cd python-client

# Single translation
python translator.py --text "Good morning" --lang English

# Interactive REPL
python translator.py --interactive
```

---

## 6. React Native app

```bash
npx react-native init DarijaTranslator
cd DarijaTranslator
npm install @react-native-async-storage/async-storage
cp ../react-native-client/App.js .

# Android
npx react-native run-android

# iOS
npx react-native run-ios
```

Update `DEFAULT_SERVER` in `App.js` to your server IP (use `10.0.2.2` for Android emulator).

---

## UML — Class Diagram (key classes)

```
RestApplication           (extends Application, @ApplicationPath("/api"))
    └─ TranslatorResource (@Path("/translate"), @RolesAllowed("translator"))
           ├─ translate(TranslationRequest) : Response   [POST]
           ├─ translateGet(text, lang)      : Response   [GET]
           └─ health()                      : Response   [GET /health]

GeminiService             (@ApplicationScoped CDI bean)
    └─ translate(text, sourceLang) : String
    └─ buildPrompt(text, lang)     : String
    └─ buildRequestBody(prompt)    : String
    └─ extractTranslation(json)    : String

TranslationRequest        (DTO — JSON body)
    + text       : String
    + sourceLang : String

CorsFilter                (implements Filter)
    + doFilter(...)
```

---

## Security notes

- Always use HTTPS in production — set `<transport-guarantee>CONFIDENTIAL</transport-guarantee>` in `web.xml`
- Store the Gemini API key as an environment variable, never in source code
- Restrict `Access-Control-Allow-Origin` to your specific extension/domain in production
- Consider token-based auth (JWT) instead of Basic Auth for stronger security

---

## Extension features

| Feature | Implementation |
|---|---|
| Side panel | `chrome.sidePanel` API (Manifest V3) |
| Right-click menu | `chrome.contextMenus` |
| Auto-paste selection | `chrome.storage.local` relay via background SW |
| Read source aloud | `SpeechSynthesisUtterance` with source language |
| Read Darija aloud | `SpeechSynthesisUtterance` with `lang: 'ar-MA'` |
| Persist settings | `chrome.storage.sync` |
