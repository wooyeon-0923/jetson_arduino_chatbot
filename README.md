# jetson_arduino_chatbot

---

# 🌟 Chatbot Project for Sensor Monitoring

## 📋 프로젝트 개요

**Grove Dust Sensor**와 **CM1106 CO2 Sensor**를 활용해 실시간 데이터를 수집하고 이를 바탕으로 **예측 모델**과 **챗봇**을 구현하는 과정을 다룹니다. 챗봇은 수집된 데이터를 바탕으로 사용자와 상호작용하며, 센서 상태와 공기질에 대한 정보를 제공합니다.

---

## 🚀 주요 목표

1. **센서 데이터 수집**:  
   - Grove Dust Sensor를 통해 미세먼지 농도를 5초 간격으로 측정.
   - CM1106 CO2 Sensor를 통해 이산화탄소 농도를 5초 간격으로 측정.

2. **데이터 저장**:  
   - 수집된 센서 데이터를 CSV 파일로 저장.
   - Python 스크립트를 활용하여 저장된 데이터를 분석.

3. **머신러닝 모델 구축**:  
   - 수집된 데이터를 기반으로 머신러닝 모델을 사용해 **미래 농도** 예측.
   - ARIMA, Random Forest Regressor, 또는 LSTM을 활용.

4. **챗봇 구현**:  
   - OpenAI API를 활용하여 챗봇 생성.
   - Function Calling 기능으로 실시간 센서 상태를 분석하고 사용자와 상호작용.

---

## ⚙️ 시스템 구성

### 🖥️ 하드웨어 구성
- **Grove Dust Sensor**
- **CM1106 CO2 Sensor**
- **Jetson Nano**
- **Arduino**

### 🛠️ 소프트웨어 구성
- **Python**
  - 데이터 처리 및 머신러닝 모델 구축.
- **Arduino**
  - 센서 데이터 수집 및 시리얼 통신.
- **OpenAI API**
  - Function Calling 기반의 챗봇 구현.

---

## 🧩 주요 기능

### 1️⃣ **센서 데이터 수집**
#### 📄 Grove Dust Sensor 코드
```cpp
int pin = 8;
unsigned long duration;
unsigned long starttime;
unsigned long sampletime_ms = 5000;
unsigned long lowpulseoccupancy = 0;
float ratio = 0;
float concentration = 0;

void setup() {
    Serial.begin(9600);
    pinMode(pin, INPUT);
    starttime = millis();
    Serial.println("time,dust_concentration,air_quality");
}

void loop() {
    duration = pulseIn(pin, LOW);
    lowpulseoccupancy += duration;

    if ((millis() - starttime) > sampletime_ms) {
        ratio = lowpulseoccupancy / (sampletime_ms * 10.0);
        concentration = 1.1 * pow(ratio, 3) - 3.8 * pow(ratio, 2) + 520 * ratio + 0.62;

        String airQuality = (concentration <= 30) ? "Good" :
                            (concentration <= 80) ? "Moderate" :
                            (concentration <= 150) ? "Unhealthy" : "Very Unhealthy";

        Serial.print(millis() / 1000); Serial.print(",");
        Serial.print(concentration); Serial.print(",");
        Serial.println(airQuality);

        lowpulseoccupancy = 0;
        starttime = millis();
    }
}
```

#### 📄 CM1106 CO2 Sensor 코드
```cpp
#include <cm1106_i2c.h>
CM1106_I2C cm1106_i2c;

void setup() {
    cm1106_i2c.begin();
    Serial.begin(9600);
    Serial.println("time,co2_concentration");
}

void loop() {
    if (cm1106_i2c.measure_result() == 0) {
        Serial.print(millis() / 1000); Serial.print(",");
        Serial.println(cm1106_i2c.co2);
    }
    delay(5000);
}
```

---

### 2️⃣ **데이터 저장**
Python 스크립트를 사용해 센서 데이터를 CSV 파일로 저장합니다.

#### 📄 Python 코드
```python
import serial
import csv

SERIAL_PORT = 'COM3'
BAUD_RATE = 9600
CSV_FILE = "sensor_data.csv"

with open(CSV_FILE, mode='w', newline='') as file:
    writer = csv.writer(file)
    writer.writerow(["Time (s)", "Dust Concentration (ug/m3)", "CO2 Concentration (ppm)"])
    with serial.Serial(SERIAL_PORT, BAUD_RATE, timeout=1) as ser:
        while True:
            line = ser.readline().decode('utf-8').strip()
            if line:
                writer.writerow(line.split(","))
```

---

### 3️⃣ **머신러닝 기반 예측**
#### 📄 ARIMA 모델
```python
from statsmodels.tsa.arima.model import ARIMA
import pandas as pd

df = pd.read_csv("sensor_data.csv")
dust_series = df['Dust Concentration (ug/m3)']

model = ARIMA(dust_series, order=(5, 1, 0))
model_fit = model.fit()
forecast = model_fit.forecast(steps=10)
print("Forecast:", forecast)
```

---

### 4️⃣ **Function Calling 기반 챗봇**
OpenAI API를 사용하여 센서 상태를 실시간으로 분석하는 챗봇을 구현합니다.

#### 📄 Function Calling 예제
```python
import openai

openai.api_key = "YOUR_API_KEY"

def analyze_sensor_data(dust, co2):
    if dust > 100 or co2 > 800:
        return "Warning: Poor air quality detected!"
    return "Air quality is normal."

functions = [
    {
        "name": "analyze_sensor_data",
        "description": "Analyzes dust and CO2 concentrations.",
        "parameters": {
            "type": "object",
            "properties": {
                "dust": {"type": "number"},
                "co2": {"type": "number"}
            },
            "required": ["dust", "co2"]
        }
    }
]

response = openai.ChatCompletion.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Dust: 120, CO2: 900"}],
    functions=functions,
    function_call={"name": "analyze_sensor_data"}
)

print(response)
```

---

## 📈 기대 효과
1. **실시간 센서 모니터링**:
   - 미세먼지 및 CO2 농도 데이터를 실시간으로 확인.
2. **미래 농도 예측**:
   - 머신러닝 모델을 활용한 향후 농도 변화 예측.
3. **사용자 친화적 인터페이스**:
   - 챗봇을 통해 공기질 상태 및 예측 결과를 대화형으로 제공.

---

## 🛠️ 향후 개선 방향
1. **더 많은 센서 추가**:
   - 온도, 습도 등 환경 데이터를 추가 수집.
2. **STT/TTS 통합**:
   - 음성으로 챗봇과 상호작용.
3. **IoT 플랫폼 연동**:
   - 클라우드 기반 데이터 저장 및 원격 모니터링.

---
