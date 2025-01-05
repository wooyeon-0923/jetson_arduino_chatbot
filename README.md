# jetson_arduino_chatbot

## 💨 Air Quality Monitoring & Chatbot Project

### **Project Overview**
This project leverages **Jetson Nano** and a **PM10 API** to measure fine dust concentrations in real time, allowing users to easily check indoor air quality through a chatbot interface.  
We utilize **Gradio UI** and **OpenAI GPT-4** to provide data-based recommendations and insights.

---

## 🎯 Key Features
1. **Comparing Sensor Data and API Data**  
   - Compare Dust Sensor readings from Jetson Nano with PM10 API data to recommend ventilation or air purifier use.
2. **Gradio UI-Based Chatbot**  
   - Analyze data in real time and provide appropriate answers to user queries.
3. **Integration with OpenAI GPT-4**  
   - Leverage OpenAI GPT-4 API to handle more complex questions and provide thorough responses.

---

## ⚙️ System Architecture

### 🖥️ Hardware Components
- **Grove Dust Sensor**
- **CM1106 CO2 Sensor**
- **Jetson Nano**
- **Arduino**

### 🛠️ Software Components
- **Python**  
  - Data processing and machine learning model development.
- **Arduino**  
  - Sensor data collection and serial communication.
- **OpenAI API**  
  - Function Calling-based chatbot implementation.

---

## 🛠️ Project Structure

### 1. Technologies Used
- **Jetson Nano**: Processes fine dust sensor data.
- **Dust Sensor**: Measures indoor fine dust concentrations in µg/m³.
- **PM10 API**: Obtains real-time PM10 data from an external API.
- **Python Libraries**:
  - `gradio`, `pandas`, `openai`, `urllib`.

### 2. Requirements
- Python 3.8+
- Internet connection (Required for PM10 API and GPT-4 API calls)

---

## 💾 Installation & Execution

1. **Set Up Python Environment**  
   Install the required libraries using the following command:
   ```bash
   pip install gradio pandas openai
   ```

2. **Configure OpenAI API Key**  
   Set the `OPENAI_API_KEY` as an environment variable or directly in the code:
   ```python
   os.environ['OPENAI_API_KEY'] = 'YOUR_OPENAI_API_KEY'
   ```

3. **Prepare Dust Sensor Data**  
   Make sure the fine dust data collected by the Jetson Nano is stored in CSV format. For example:
   ```csv
   Timestamp,Dust Concentration (ug/m3)
   2024-12-01 10:00:00,35.2
   2024-12-01 11:00:00,40.5
   ```

4. **Run the Code**  
   Execute the script using:
   ```bash
   python app.py
   ```

---

## 💻 Main Code Explanation

### 1. PM10 API Data Retrieval
Fetch PM10 data via API and convert it into a DataFrame:
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

### 2. Reading Jetson Nano Sensor Data
Retrieve the latest Dust Sensor reading from a CSV file:
```python
df_dust = pd.read_csv('dust_sensor_data.csv')
last_sensor_value = float(df_dust['Dust Concentration (ug/m3)'].iloc[-1])
```

### 3. OpenAI API & Gradio UI Integration
A Gradio UI allows users to input questions, and the OpenAI API is called to generate responses:
```python
with gr.Blocks() as demo:
    chatbot = gr.Chatbot(label="Chat Window")
    user_textbox = gr.Textbox(label="Input")
    user_textbox.submit(process, [user_textbox, chatbot], [user_textbox, chatbot])

demo.launch(share=True, debug=True)
```

### 4. Complete Code
Below is the entire Python script:

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
## 💻 Demonstration Result

https://github.com/user-attachments/assets/8bf758d6-a6f7-4aa3-8641-e2c43969a85d

---

## 📈 Expected Benefits
1. **Real-Time Sensor Monitoring**  
   - Track real-time fine dust and CO2 concentrations.
2. **Future Concentration Prediction**  
   - Use machine learning models to predict future dust concentration changes.
3. **User-Friendly Interface**  
   - Provide air quality status and predictions through a conversational chatbot.

---

## 🛠️ Future Improvements
1. **Additional Sensors**  
   - Collect more environmental data such as temperature and humidity.
2. **STT/TTS Integration**  
   - Enable voice-based interactions with the chatbot.
3. **IoT Platform Integration**  
   - Store data in the cloud and support remote monitoring.
