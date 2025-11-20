# ⏰ Projeto IoT GS – Sistema de Monitoramento de estresse

### 📡 ESP32 • MQTT • Dashboard em Python • Global Solution

---

## 🎯 **Visão Geral do Projeto**

Este projeto IoT foi desenvolvido para monitorar o nível de bem-estar dos funcionários em home-office em **tempo real**, utilizando um **ESP32**, comunicação via **MQTT** e um **dashboard profissional em Python**.

Ele foi criado para atender a saúde e prevenção de queda de performance e burnout no ambiente profissional, oferecendo uma ferramenta prática, moderna e eficiente de monitoreamento.

---

# Link do video de apresentação do projeto


---

## 🚀 Funcionalidades Principais
### ✔️ Dispositivo IoT (ESP32)

Envia informações sobre:

“Estou bem”

“Cansado”

“Muito cansado”

Detecta padrões de cansaço e sugere pausas automaticamente.

Envia dados via MQTT para o Dashboard.

Mostra mensagens em um LCD I2C.

Suporta modo de teste com ciclos acelerados.

### ✔️ Dashboard Web (Python + Dash)

Interface moderna com estética de escritório (tons escuros e amadeirados).

Exibe três contadores com o número de respostas por categoria.

Caixa de sugestões de pausa recebidas em tempo real.

Botões:

Forçar pausa

Limpar dados

Responsivo, rápido e organizado.

Tudo isso através de botões físicos conectados ao ESP32, que envia os dados instantaneamente para o dashboard via MQTT.

---

## 🧩 **Arquitetura Resumida**
```cpp
ESP32 → MQTT Broker (HiveMQ) → Dashboard Python (Dash) → Visualização e Controle
```
* O ESP32 publica mensagens em tópicos específicos
* O Dashboard assina esses tópicos e atualiza a interface automaticamente
* O usuário visualiza tudo em tempo real em um layout estilizado

---

## 📝 Esquemático de Ligação
<img width="1919" height="956" alt="image" src="https://github.com/user-attachments/assets/7f020f7f-79a4-4096-b76b-694bb709e3b1" />

---

## ⚙️ Configuração do Ambiente
1. Instalação do Arduino IDE

- Baixar e instalar Arduino IDE.

- Adicionar suporte ao ESP32:

  - Arquivo > Preferências > URL adicionais de placas:

      https://dl.espressif.com/dl/package_esp32_index.json


  - Ferramentas > Placa > Gerenciador de Placas → Pesquisar e instalar ESP32.

2. Bibliotecas Necessárias

Instale as seguintes bibliotecas via Gerenciador de Bibliotecas:

- Wire
  
- LiquidCrystal_I2C
  
- ArduinoJson

- WiFi

- PubSubClient (para MQTT)

---

## 📲 Configuração do MyMQTT
O ESP32 **publica** mensagens nesses tópicos sempre que um botão é pressionado.
O dashboard em Python **assina** esses mesmos tópicos e atualiza a interface em tempo real.


Por ser um broker público, não é necessário criar conta ou gerar token. Basta conectar com o endereço e porta acima.

---

# 🔧 **Código Fonte – ESP32**
```cpp
/* ESP32 - Detector de nível de estresse (Wokwi) - LCD I2C
   Versão com DEBUG/robustez MQTT
*/

#include <WiFi.h>
#include <PubSubClient.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <ArduinoJson.h>

// -------- CONFIG ----------
const char* ssid = "Wokwi-GUEST";
const char* password = "";

// MQTT broker público (mude se quiser)
const char* mqtt_server = "broker.hivemq.com";
const int mqtt_port = 1883;

// IDs / tópicos
const char* deviceIdBase = "esp32_stress_001";
char deviceId[40];
const char* topic_status = "gs-edge/stress/status";
const char* topic_suggestion = "gs-edge/stress/suggestion";
const char* topic_cmd = "gs-edge/stress/cmd";

// Hardware pins
const int BTN1_PIN = 12; // bem
const int BTN2_PIN = 13; // cansado
const int BTN3_PIN = 14; // muito cansado
const int BUZZER_PIN = 25;
const int LEDR = 16, LEDG = 17, LEDB = 27;

// LCD I2C (ajuste o endereço se necessário: 0x27 ou 0x3F)
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Timing
#define WINDOW_SIZE 6
bool SIM_FAST = true;
unsigned long interval_hour_ms = 60UL * 60UL * 1000UL;
unsigned long interval_testing_ms = 60UL * 1000UL;
unsigned long REPORT_DEADLINE_FACTOR = 150;

// debounce
unsigned long lastDebounce[3] = {0,0,0};
const unsigned long DEBOUNCE_MS = 50;

// storage
int responses[WINDOW_SIZE];
int respIndex = 0;
int respCount = 0;

unsigned long lastHourlyMillis = 0;
unsigned long interval_ms() { return SIM_FAST ? interval_testing_ms : interval_hour_ms; }

// WiFi + MQTT
WiFiClient espClient;
PubSubClient client(espClient);

// helper: buzzer beep
void beep(int ms) {
  tone(BUZZER_PIN, 2000);
  delay(ms);
  noTone(BUZZER_PIN);
}

// helper: imprime no LCD (até 2 linhas, 16 chars cada)
void lcdPrint(const String &line1, const String &line2 = "") {
  lcd.clear();
  String l1 = line1; String l2 = line2;
  if (l1.length() > 16) l1 = l1.substring(0,16);
  if (l2.length() > 16) l2 = l2.substring(0,16);
  lcd.setCursor(0,0);
  lcd.print(l1);
  lcd.setCursor(0,1);
  lcd.print(l2);
  // também log no serial pra debug
  Serial.print("[LCD] ");
  Serial.print(l1);
  if (l2.length()) {
    Serial.print(" / ");
    Serial.print(l2);
  }
  Serial.println();
}

// MQTT publish helper (JSON) — agora garante conexão
bool safePublish(const char* topic, const char* payload) {
  if (!client.connected()) {
    Serial.println("MQTT nao conectado — tentando reconectar...");
    // tenta reconectar rápido (não bloqueante aqui além do reconnect)
    unsigned long t0 = millis();
    while (!client.connected() && millis() - t0 < 5000) {
      if (client.connect(deviceId)) {
        Serial.println("Reconectado ao MQTT no safePublish");
        client.subscribe(topic_cmd);
        break;
      }
      delay(200);
    }
  }
  if (client.connected()) {
    boolean ok = client.publish(topic, payload);
    Serial.print("publish to "); Serial.print(topic); Serial.print(" -> "); Serial.println(ok ? "OK" : "FAIL");
    return ok;
  } else {
    Serial.println("Falha: ainda sem conexao MQTT. Payload descartado.");
    return false;
  }
}

void publishStatus() {
  StaticJsonDocument<256> doc;
  doc["device"] = deviceId;
  JsonArray arr = doc.createNestedArray("last_responses");
  for (int i=0;i<respCount;i++) {
    int idx = (respIndex - respCount + i + WINDOW_SIZE) % WINDOW_SIZE;
    arr.add(responses[idx]);
  }
  float avg = 0;
  for (int i=0;i<respCount;i++) {
    int idx = (respIndex - respCount + i + WINDOW_SIZE) % WINDOW_SIZE;
    avg += responses[idx];
  }
  if (respCount>0) avg /= respCount;
  doc["avg_score"] = avg;
  char buf[256];
  size_t n = serializeJson(doc, buf);
  safePublish(topic_status, buf);
}

void suggestBreak(const char* reason) {
  StaticJsonDocument<200> doc;
  doc["device"] = deviceId;
  doc["suggestion"] = "PAUSA_RECOMENDADA";
  doc["reason"] = reason;
  char buf[200];
  size_t n = serializeJson(doc, buf);
  safePublish(topic_suggestion, buf);
  lcdPrint("SUGESTAO: PAUSA", reason);
  for (int i=0;i<2;i++) {
    beep(250);
    digitalWrite(LEDR, HIGH); delay(150); digitalWrite(LEDR, LOW); delay(150);
  }
}

void decisionLogic() {
  if (respCount == 0) return;
  float avg = 0;
  for (int i=0;i<respCount;i++) {
    int idx = (respIndex - respCount + i + WINDOW_SIZE) % WINDOW_SIZE;
    avg += responses[idx];
  }
  avg /= respCount;
  if (avg >= 1.2) { suggestBreak("media_alta_cansaco"); return; }
  int consec2 = 0;
  for (int i=respCount-1;i>=0 && consec2<2;i--) {
    int idx = (respIndex - 1 - (respCount-1-i) + WINDOW_SIZE) % WINDOW_SIZE;
    int v = responses[idx];
    if (v==2) consec2++; else break;
  }
  if (consec2 >= 2) { suggestBreak("consecutivos_muito_cansado"); return; }
}

void registerResponse(int score) {
  responses[respIndex] = score;
  respIndex = (respIndex + 1) % WINDOW_SIZE;
  if (respCount < WINDOW_SIZE) respCount++;
  publishStatus();
  decisionLogic();
  String s;
  if (score==0) s="Estou bem";
  else if (score==1) s="Cansado";
  else s="Muito cansado";
  lcdPrint("Resposta gravada:", s);
  beep(120);
}

void callback(char* topic, byte* payload, unsigned int length) {
  String t = String(topic);
  String msg;
  for (unsigned int i=0;i<length;i++) msg += (char)payload[i];
  Serial.print("MQTT RX: topic="); Serial.print(t); Serial.print(" payload="); Serial.println(msg);
  if (t == topic_cmd) {
    if (msg.indexOf("FORCE_BREAK") >= 0) {
      suggestBreak("dashboard_forcado");
    }
  }
}

void reconnect() {
  if (client.connected()) return;
  Serial.print("Tentando conectar ao MQTT...");
  int tries = 0;
  while (!client.connected() && tries < 10) {
    // usa deviceId já definido
    if (client.connect(deviceId)) {
      Serial.println("OK");
      client.subscribe(topic_cmd);
      return;
    } else {
      Serial.print(".");
      delay(500);
      tries++;
    }
  }
  if (!client.connected()) {
    Serial.println(" falha conectar MQTT (timeout).");
  }
}

void setup() {
  Serial.begin(115200);
  // cria deviceId único -> deviceIdBase + random
  uint32_t r = esp_random() & 0xFFFF;
  snprintf(deviceId, sizeof(deviceId), "%s_%04X", deviceIdBase, (unsigned)r);
  Serial.print("DeviceId: "); Serial.println(deviceId);

  pinMode(BTN1_PIN, INPUT_PULLUP);
  pinMode(BTN2_PIN, INPUT_PULLUP);
  pinMode(BTN3_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LEDR, OUTPUT);
  pinMode(LEDG, OUTPUT);
  pinMode(LEDB, OUTPUT);

  Wire.begin(); // SDA=21, SCL=22 no ESP32
  lcd.init(); lcd.backlight();
  lcdPrint("Stress Monitor", "Inicializando...");

  // WiFi
  WiFi.begin(ssid, password);
  unsigned long start = millis();
  while (WiFi.status() != WL_CONNECTED && millis()-start < 15000) {
    delay(500);
    Serial.print(".");
  }
  Serial.println();
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("WiFi conectado");
    String ip = WiFi.localIP().toString();
    lcdPrint("WiFi conectado", ip);
  } else {
    lcdPrint("WiFi falhou", "Continuando offline");
  }

  client.setServer(mqtt_server, mqtt_port);
  client.setCallback(callback);

  // se já estamos com WiFi, tenta conectar MQTT imediatamente
  if (WiFi.status() == WL_CONNECTED) reconnect();

  lastHourlyMillis = millis();
  if (SIM_FAST) lcdPrint("Modo TESTE ON", "Intervalo curto (1min)");
  else lcdPrint("Modo NORMAL", "Intervalo 1 hora");
}

void loop() {
  if (WiFi.status() == WL_CONNECTED && !client.connected()) reconnect();
  client.loop();

  unsigned long now = millis();

  if (digitalRead(BTN1_PIN) == LOW) {
    if (millis() - lastDebounce[0] > DEBOUNCE_MS) {
      lastDebounce[0] = millis();
      registerResponse(0);
    }
    while (digitalRead(BTN1_PIN) == LOW) delay(10);
  }
  if (digitalRead(BTN2_PIN) == LOW) {
    if (millis() - lastDebounce[1] > DEBOUNCE_MS) {
      lastDebounce[1] = millis();
      registerResponse(1);
    }
    while (digitalRead(BTN2_PIN) == LOW) delay(10);
  }
  if (digitalRead(BTN3_PIN) == LOW) {
    if (millis() - lastDebounce[2] > DEBOUNCE_MS) {
      lastDebounce[2] = millis();
      registerResponse(2);
    }
    while (digitalRead(BTN3_PIN) == LOW) delay(10);
  }

  if (now - lastHourlyMillis >= interval_ms()) {
    lastHourlyMillis = now;
    lcdPrint("Hora de reportar", "Pressione um botao");
    StaticJsonDocument<200> doc;
    doc["device"] = deviceId;
    doc["event"] = "hora_de_reportar";
    char buf[200];
    size_t n = serializeJson(doc, buf);
    safePublish(topic_status, buf);
    unsigned long deadline = interval_ms() * REPORT_DEADLINE_FACTOR / 100;
    unsigned long startWait = millis();
    bool responded = false;
    int initialRespCount = respCount;
    while (millis() - startWait < deadline) {
      if (respCount > initialRespCount) { responded = true; break; }
      delay(100);
      client.loop();
    }
    if (!responded) {
      registerResponse(1);
      StaticJsonDocument<200> doc2;
      doc2["device"] = deviceId;
      doc2["auto_assumed"] = "cansado";
      char buf2[200];
      size_t n2 = serializeJson(doc2, buf2);
      safePublish(topic_status, buf2);
      decisionLogic();
    }
  }

  delay(20);
}
```
---

# 📟 **Código Fonte – Diagram.json (conexões)**
```json
{
  "version": 1,
  "author": "Guilherme",
  "editor": "wokwi",
  "parts": [
    { "type": "board-esp32-devkit-c-v4", "id": "esp", "top": -28.8, "left": -14.36, "attrs": {} },
    {
      "type": "wokwi-pushbutton",
      "id": "btn1",
      "top": -51.4,
      "left": 134.4,
      "attrs": { "color": "green", "xray": "1", "label": "BTN1 - Estou bem" }
    },
    {
      "type": "wokwi-pushbutton",
      "id": "btn2",
      "top": 15.8,
      "left": 211.2,
      "attrs": { "color": "yellow", "xray": "1", "label": "BTN2 - Cansado" }
    },
    {
      "type": "wokwi-pushbutton",
      "id": "btn3",
      "top": 73.4,
      "left": 134.4,
      "attrs": { "color": "red", "xray": "1", "label": "BTN3 - Muito cansado" }
    },
    {
      "type": "wokwi-buzzer",
      "id": "buzzer",
      "top": -218.4,
      "left": -7.8,
      "attrs": { "label": "Buzzer" }
    },
    {
      "type": "wokwi-lcd1602",
      "id": "lcd1",
      "top": -156.8,
      "left": 303.2,
      "attrs": { "pins": "i2c" }
    }
  ],
  "connections": [
    [ "esp:TX", "$serialMonitor:RX", "", [] ],
    [ "esp:RX", "$serialMonitor:TX", "", [] ],
    [ "btn1:2.r", "esp:GND.1", "black", [ "h0" ] ],
    [ "btn2:2.r", "esp:GND.1", "black", [ "h0" ] ],
    [ "btn3:2.r", "esp:GND.1", "black", [ "h0" ] ],
    [ "btn1:1.l", "esp:12", "green", [ "h0" ] ],
    [ "btn2:1.l", "esp:13", "gold", [ "h0" ] ],
    [ "btn3:1.l", "esp:14", "red", [ "h0" ] ],
    [ "buzzer:2", "esp:25", "red", [ "h0" ] ],
    [ "buzzer:1", "esp:GND.1", "black", [ "h0" ] ],
    [ "lcd1:SDA", "esp:21", "green", [ "h0" ] ],
    [ "lcd1:SCL", "esp:22", "green", [ "h0" ] ],
    [ "lcd1:VCC", "esp:3V3", "red", [ "h0" ] ],
    [ "lcd1:GND", "esp:GND.1", "black", [ "h0" ] ]
  ],
  "dependencies": {}
}
```
---

## 📊 Dashboard em Python (Monitoramento em Tempo Real)
Para complementar o projeto e permitir a visualização dos dados recebidos tempo real, foi desenvolvido um Dashboard interativo utilizando Python, com as bibliotecas Dash e MQTT.

Este exibe três caixas contendo os feedbacks enviados pelos funcionários, sendo elas "estou bem", "cansado" e "muito cansado".

---
## 📝 DASHBOARD
<img width="1511" height="786" alt="image" src="https://github.com/user-attachments/assets/a5a5660f-8cc3-4aac-aeda-4bafacfbf3ee" />

---

# 🖥️ **Código Fonte – Dashboard em Python (Dash / MQTT)**

```python
# main.py
"""
Dashboard - Estilo escritório escuro (Dash)
- Usa CSS em ./assets/style.css
- Exibe três caixas com contadores: Estou bem / Cansado / Muito cansado
- Mantém sugestão de pausa e botão Forçar pausa
- Atualiza via MQTT (broker público por padrão)
"""

import threading, time, json
from collections import deque
import dash
from dash import dcc, html, Output, Input, State
import paho.mqtt.client as mqtt
from threading import Lock

# ---------- CONFIG ----------
BROKER = "broker.hivemq.com"
PORT = 1883
TOPIC_STATUS = "gs-edge/stress/status"
TOPIC_SUG = "gs-edge/stress/suggestion"
TOPIC_CMD = "gs-edge/stress/cmd"

# counters por categoria (0=bem,1=cansado,2=muito cansado)
counters = {0: 0, 1: 0, 2: 0}
last_suggestion = {"time": None, "reason": None}

lock = Lock()

# ---------- MQTT callbacks ----------
def on_connect(client, userdata, flags, rc):
    print("MQTT (dash) conectado rc=", rc)
    client.subscribe(TOPIC_STATUS)
    client.subscribe(TOPIC_SUG)
    print("Subscreveu:", TOPIC_STATUS, TOPIC_SUG)

def on_message(client, userdata, msg):
    try:
        payload = json.loads(msg.payload.decode())
    except Exception as e:
        print("Erro parse JSON:", e, "raw:", msg.payload)
        return
    print("MQTT (dash) RX:", msg.topic, payload)
    if msg.topic == TOPIC_STATUS:
        # payload esperado: {"device":"...","last_responses":[...],"avg_score":...}
        lr = payload.get("last_responses", [])
        if lr:
            last = lr[-1]
            try:
                v = int(last)
                if v in (0,1,2):
                    with lock:
                        counters[v] += 1
                else:
                    print("Valor fora do esperado:", v)
            except Exception as e:
                print("Nao foi possivel converter valor:", last, e)
    elif msg.topic == TOPIC_SUG:
        last_suggestion["time"] = time.time()
        last_suggestion["reason"] = payload.get("reason", "")

def mqtt_thread():
    client = mqtt.Client()
    client.on_connect = on_connect
    client.on_message = on_message
    try:
        print("Conectando ao broker", BROKER)
        client.connect(BROKER, PORT, 60)
    except Exception as e:
        print("Erro conectando ao broker:", e)
        return
    client.loop_forever()

# inicia thread MQTT
t = threading.Thread(target=mqtt_thread, daemon=True)
t.start()

# ---------- DASH app ----------
# Dash carrega automaticamente arquivos CSS colocados em ./assets
app = dash.Dash(__name__, suppress_callback_exceptions=True)
server = app.server

# Layout com classes CSS correspondentes a assets/style.css
app.layout = html.Div(className="app-container", children=[
    # Header
    html.Div(className="header-row", children=[
        html.Div(className="header-left", children=[
            html.H1("Monitor de Estresse", className="header-title"),
            html.Div("Painel de acompanhamento — home office", className="header-sub")
        ]),
        html.Div(className="header-right", id="header-right", children="Status: verificando..."),
    ]),

    # Main row
    html.Div(className="main-row", children=[
        html.Div(className="left-col", children=[
            html.Div(className="card", children=[
                html.Div("Estou bem", className="counter-label"),
                html.Div(id="count-bem", className="counter-value")
            ]),
            html.Div(className="card", children=[
                html.Div("Cansado", className="counter-label"),
                html.Div(id="count-cansado", className="counter-value")
            ]),
            html.Div(className="card", children=[
                html.Div("Muito cansado", className="counter-label"),
                html.Div(id="count-muito", className="counter-value")
            ])
        ]),

        html.Div(className="right-col", children=[
            html.Div(className="card suggestion-box", children=[
                html.Div("Sugestão de pausa", className="counter-label"),
                html.Div(id="last-suggestion", style={'marginTop':'10px'})
            ]),
            html.Div(className="card", style={'marginTop':'18px'}, children=[
            html.Button("Forçar pausa (enviar comando)", id='force-break', n_clicks=0, className="btn-force"),
            html.Div(id='send-status', className="send-status"),
            html.Button("Limpar dados", id='reset-dashboard', n_clicks=0, className="btn-reset", style={'marginTop':'15px'}),
            html.Div(id="reset-status", className="send-status")
        ])

        ])
    ]),

    # Hidden interval para atualizar contadores e status
    dcc.Interval(id="interval", interval=1500, n_intervals=0)
])

# ---------- Callbacks ----------
@app.callback(
    Output("count-bem","children"),
    Output("count-cansado","children"),
    Output("count-muito","children"),
    Output("header-right","children"),
    Input("interval","n_intervals")
)
def update_counts(n):
    with lock:
        b = counters.get(0,0)
        c = counters.get(1,0)
        m = counters.get(2,0)
    # exibe também um pequeno status se temos contadores (indica dados chegando)
    status_text = f"Broker: {BROKER} — total eventos: {b + c + m}"
    return f"{b:,}", f"{c:,}", f"{m:,}", status_text

@app.callback(Output("last-suggestion","children"), Input("interval","n_intervals"))
def update_suggestion(n):
    if last_suggestion["time"]:
        t = time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(last_suggestion["time"]))
        reason = last_suggestion.get("reason","")
        return html.Div([
            html.Div(t, className="suggestion-time"),
            html.Div(f"Motivo: {reason}", className="suggestion-reason")
        ])
    return html.Div("Nenhuma sugestão registrada", className="suggestion-time")

@app.callback(Output("send-status","children"), Input("force-break","n_clicks"))
def force_break(n):
    if n > 0:
        try:
            client = mqtt.Client()
            client.connect(BROKER, PORT, 60)
            payload = json.dumps({"cmd":"FORCE_BREAK"})
            client.publish(TOPIC_CMD, payload)
            client.disconnect()
            print("Enviado FORCE_BREAK via MQTT")
            return "Comando enviado!"
        except Exception as e:
            print("Erro ao enviar comando:", e)
            return f"Erro: {e}"
    return "Pronto."

@app.callback(
    Output("reset-status", "children"),
    Input("reset-dashboard", "n_clicks")
)
def reset_dashboard(n):
    if n and n > 0:
        with lock:
            counters[0] = 0
            counters[1] = 0
            counters[2] = 0
        last_suggestion["time"] = None
        last_suggestion["reason"] = None
        return "Dashboard limpo!"
    return ""

# ---------- run ----------
if __name__ == "__main__":
    print("Iniciando dashboard - estilo escritório escuro...")
    # host 0.0.0.0 para acessar pela rede local, use localhost se preferir
    app.run(debug=True, host="0.0.0.0", port=8050)
```
---

# 🎨  **Código Fonte – CSS (estilização)**
```css
/* assets/style.css - estilo escuro amadeirado para o Dashboard
   - Salve este arquivo em ./assets/style.css no diretório do seu projeto Dash
   - Dash carrega automaticamente arquivos na pasta "assets" */

:root{
  --bg-top: #231812;
  --bg-mid: #2f1d13;
  --bg-bot: #3b2a1f;
  --card-top: #3a2a20;
  --card-bot: #2b1b14;
  --accent: #9b5f2b;
  --muted: #e6d6c6;
  --text: #ffffff;
}

html, body {
  height: 100%;
  margin: 0;
  padding: 0;
  background: linear-gradient(180deg, var(--bg-top) 0%, var(--bg-mid) 40%, var(--bg-bot) 100%);
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  color: var(--text);
}

/* Container central */
.app-container {
  max-width: 1100px;
  margin: 20px auto;
  padding: 18px;
}

/* Header */
.header-row{
  display:flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  padding-bottom: 18px;
}
.header-left{
  text-align: left;
}
.header-title{
  font-size: 22px;
  font-weight: 700;
  margin: 0;
  color: var(--text);
}
.header-sub{
  font-size: 12px;
  color: var(--muted);
  margin-top: 6px;
}
.header-right{
  color: var(--muted);
  font-size: 13px;
}

/* Layout columns */
.main-row{
  display:flex;
  gap: 20px;
  align-items: flex-start;
}
.left-col{
  width: 320px;
  display:flex;
  flex-direction: column;
  gap: 14px;
}
.right-col{
  flex: 1;
  min-width: 280px;
}

/* Card */
.card{
  background: linear-gradient(180deg, var(--card-top), var(--card-bot));
  border-radius: 10px;
  padding: 18px;
  box-shadow: 0 6px 18px rgba(0,0,0,0.35);
  color: var(--text);
  text-align: center;
}

.counter-label{
  font-size: 14px;
  color: var(--text);
  opacity: 0.95;
}
.counter-value{
  font-size: 34px;
  font-weight: 700;
  margin-top: 8px;
}

/* Suggestion area */
.suggestion-box{
  min-height: 120px;
  text-align: left;
}
.suggestion-time{
  font-size: 12px;
  color: var(--muted);
}
.suggestion-reason{
  margin-top: 8px;
  font-weight: 600;
}

/* Button */
.btn-force{
  width: 100%;
  padding: 12px;
  border-radius: 8px;
  background: var(--accent);
  color: white;
  border: none;
  font-weight: 600;
  cursor: pointer;
}
.btn-force:hover{ filter:brightness(1.05); }

/* small text under button */
.send-status{ font-size: 13px; color: var(--muted); margin-top: 12px; }

/* Responsividade */
@media (max-width: 900px){
  .main-row{ flex-direction: column; }
  .left-col{ width: 100%; }
  .right-col{ width: 100%; }
}
.btn-reset {
    background-color: #6b4f3a;
    color: white;
    border: none;
    padding: 10px 16px;
    font-size: 15px;
    width: 100%;
    border-radius: 10px;
    cursor: pointer;
    margin-top: 10px;
}

.btn-reset:hover {
    background-color: #8a6a50;
}
```
---

## 💻 Tecnologias Utilizadas

Dash → Criação da interface e dos painéis interativos.

CSS → Estilização com tema escuro/amadeirado e design responsivo.

Paho-MQTT → Comunicação com o broker MQTT em tempo real.

Threading e JSON → Processamento paralelo das mensagens e estruturação dos dados recebidos.

## 🔌 Execução do Dashboard

Certifique-se de ter o Python 3 instalado no sistema.

Instale as dependências necessárias executando no terminal:

pip install dash plotly dash-bootstrap-components paho-mqtt


Verifique se o ESP32 já está publicando dados no broker MQTT.

Salve o arquivo do dashboard como main.py.

Execute o dashboard com o comando:

py main.py


Após a inicialização, o terminal exibirá uma mensagem semelhante a:

Running on http://127.0.0.1:8050/


Acesse o endereço no navegador (geralmente http://localhost:8050) para visualizar o dashboard em tempo real.

---

## 🧪 Testes Realizados

- ✅ Conexão WiFi estável no ESP32.

- ✅ Publicação periódica de dados dos sensores no broker MQTT.

- ✅ Visualização dos valores no app MyMQTT em tempo real.

---

## 📚 Referências
- ESP32 — Documentação oficial: https://www.espressif.com

---

# 🏁 Conclusão
Este projeto demonstra de forma prática como:

- Integrar componentes com um microcontrolador ESP32;

- Utilizar protocolos IoT (MQTT) para comunicação em tempo real;

- Visualizar dados e atuar remotamente usando um aplicativo móvel.

A base desenvolvida pode ser facilmente expandida para incluir divisão por funcionário, tarefas concluídas, metas à alcancar e muito mais.

---

## 👥 Projeto IoT desenvolvido por:
  ## Guilherme Eduardo de Lima
  ## Guilherme de Paula
  ## Enzo de Faria
  ## Matheus Gomes



