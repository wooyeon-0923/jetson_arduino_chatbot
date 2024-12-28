# jetson_arduino_chatbot

# 💨 공기질 측정 및 챗봇 프로젝트

## 📖 프로젝트 개요
이 프로젝트는 **Jetson Nano**와 **PM10 API**를 활용하여 실시간으로 미세먼지 농도를 측정하고, 챗봇 인터페이스를 통해 사용자가 공기질 상태를 쉽게 확인할 수 있도록 합니다.  
Gradio UI와 OpenAI GPT-4를 사용하여 데이터 기반 조언을 제공합니다.

---

## 🎯 주요 기능
1. **센서 데이터와 API 데이터 비교**:
   - Jetson Nano의 Dust Sensor와 PM10 API 데이터를 비교하여 환기 또는 공기청정기 사용 권장.
2. **Gradio UI 기반 챗봇**:
   - 사용자의 질문에 대해 실시간으로 데이터를 분석하고 적절한 답변 제공.
3. **OpenAI GPT-4와 연동**:
   - OpenAI GPT-4 API를 활용하여 복잡한 질문에도 답변 가능.

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

## 🛠️ 프로젝트 구성
### 1. 사용된 기술
- **Jetson Nano**: 미세먼지 센서 데이터 처리.
- **Dust Sensor**: 실내 미세먼지 농도를 µg/m³ 단위로 측정.
- **PM10 API**: 외부 API를 통해 실시간 PM10 데이터를 가져옴.
- **Python 라이브러리**:
  - `gradio`, `pandas`, `openai`, `urllib`.

### 2. 설치 요구사항
- Python 3.8 이상
- 인터넷 연결 (PM10 API 및 GPT-4 API 호출을 위해 필요)

---

## 💾 설치 및 실행 방법

1. **Python 환경 설정**:
   아래 명령어로 필요한 라이브러리를 설치합니다.
   ```bash
   pip install gradio pandas openai
   ```

2. **OpenAI API 키 설정**:
   `OPENAI_API_KEY` 환경 변수를 설정하거나 코드 내에서 직접 입력합니다.
   ```python
   os.environ['OPENAI_API_KEY'] = 'YOUR_OPENAI_API_KEY'
   ```

3. **Dust Sensor 데이터 준비**:
   Jetson Nano에서 수집한 미세먼지 데이터는 CSV 형식으로 저장해야 합니다. 예:
   ```csv
   Timestamp,Dust Concentration (ug/m3)
   2024-12-01 10:00:00,35.2
   2024-12-01 11:00:00,40.5
   ```

4. **코드 실행**:
   아래 명령어로 스크립트를 실행합니다.
   ```bash
   python app.py
   ```

---

## 💻 주요 코드 설명

### 1. PM10 API 데이터 호출
PM10 API에서 데이터를 가져와 DataFrame으로 변환:
```python
url = 'https://apihub.kma.go.kr/api/typ01/url/kma_pm10.php?tm1=202412112020&tm2=202412122020&authKey=8diTRQm4TEKYk0UJuOxCsg'
with urlopen(url) as f:
    html = f.read().decode('euc-kr')

lines = html.split('\n')
filtered_lines = [line for line in lines if ',   132,' in line]

columns = ["TM", "STN", "PM10", "FLAG", " ", "MQC"]
data = []
for line in filtered_lines:
    values = line.split(',')
    data.append([value.strip() for value in values])

df = pd.DataFrame(data, columns=columns[:len(data[0])])
df.drop(['STN', 'FLAG', ' ', 'MQC'], axis=1, inplace=True)
last_API_value = float(df['PM10'].iloc[-1])
```

### 2. Jetson Nano 센서 데이터 읽기
CSV 파일에서 Dust Sensor의 최신 데이터를 읽어옴:
```python
df_dust = pd.read_csv('dust_sensor_data.csv')
last_sensor_value = float(df_dust['Dust Concentration (ug/m3)'].iloc[-1])
```

### 3. OpenAI API와 Gradio UI 연동
Gradio UI를 통해 사용자가 질문을 입력하고, OpenAI API를 호출해 답변을 생성:
```python
with gr.Blocks() as demo:
    chatbot = gr.Chatbot(label="Chat Window")
    user_textbox = gr.Textbox(label="Input")
    user_textbox.submit(process, [user_textbox, chatbot], [user_textbox, chatbot])

demo.launch(share=True, debug=True)
```

### 4. 전체 코드
아래는 전체 Python 코드입니다:

```python
import gradio as gr
import random
import os
from openai import OpenAI
import pandas as pd
from urllib.request import urlopen

# Set OpenAI Key
os.environ['OPENAI_API_KEY'] = 'YOUR_OPENAI_API_KEY'
OpenAI.api_key = os.getenv("OPENAI_API_KEY")

# Retrieve API data
url = 'https://apihub.kma.go.kr/api/typ01/url/kma_pm10.php?tm1=202412112020&tm2=202412122020&authKey=8diTRQm4TEKYk0UJuOxCsg'
with urlopen(url) as f:
    html = f.read().decode('euc-kr')

lines = html.split('\n')
filtered_lines = [line for line in lines if ',   132,' in line]

columns = ["TM", "STN", "PM10", "FLAG", " ", "MQC"]
data = []
for line in filtered_lines:
    values = line.split(',')
    data.append([value.strip() for value in values])

df = pd.DataFrame(data, columns=columns[:len(data[0])])
df.drop(['STN', 'FLAG', ' ', 'MQC'], axis=1, inplace=True)
last_API_value = float(df['PM10'].iloc[-1])

# Retrieve Jetson Nano sensor data
df_dust = pd.read_csv('dust_sensor_data.csv')
last_sensor_value = float(df_dust['Dust Concentration (ug/m3)'].iloc[-1])

# Define OpenAI functions
def get_last_sensor_value():
    return {"last_sensor_value": last_sensor_value}

def get_last_api_value():
    return {"last_api_value": last_API_value}

functions = [
    {
        "name": "get_last_sensor_value",
        "description": "Returns the latest dust concentration value from the Jetson Nano sensor",
        "parameters": {"type": "object", "properties": {}, "required": []}
    },
    {
        "name": "get_last_api_value",
        "description": "Returns the latest PM10 value from the API",
        "parameters": {"type": "object", "properties": {}, "required": []}
    }
]

def process(user_message, chat_history):
    if "대전 날씨" in user_message:
        if last_sensor_value > last_API_value:
            ai_message = (f"현재 젯슨 나노 측정 값({last_sensor_value} ug/m3)가 "
                          f"API PM10 값({last_API_value} ug/m3)보다 높습니다. 환기를 시키세요.")
        else:
            ai_message = (f"현재 젯슨 나노 측정 값({last_sensor_value} ug/m3)가 "
                          f"API PM10 값({last_API_value} ug/m3)보다 낮거나 같습니다. 공기청정기를 사용하세요.")
        chat_history.append((user_message, ai_message))
        return "", chat_history

    client = OpenAI()
    completion = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "system", "content": "You are a helpful assistant."},
                  {"role": "user", "content": user_message}],
        functions=functions,
        function_call="auto"
    )
    response = completion.choices[0].message

    if "function_call" in response:
        func_name = response["function_call"]["name"]
        func_result = (get_last_sensor_value() if func_name == "get_last_sensor_value" else get_last_api_value())
        completion = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "system", "content": "You are a helpful assistant."},
                      {"role": "user", "content": user_message},
                      {"role": "function", "name": func_name, "content": str(func_result)}]
        )
        ai_message = completion.choices[0].message.content
    else:
        ai_message = response["content"]

    chat_history.append((user_message, ai_message))
    return "", chat_history

with gr.Blocks() as demo:
    chatbot = gr.Chatbot(label="Chat Window")
    user_textbox = gr.Textbox(label="Input")
    user_textbox.submit(process, [user_textbox, chatbot], [user_textbox, chatbot])

demo.launch(share=True, debug=True)
```
---


https://github.com/user-attachments/assets/8bf758d6-a6f7-4aa3-8641-e2c43969a85d


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
