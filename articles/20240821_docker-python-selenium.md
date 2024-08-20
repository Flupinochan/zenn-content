---
title: "DockerでPython Selenium環境のセットアップ"
emoji: "🙆"
type: "tech"
topics: ["docker", "selenium", "python"]
published: true
---

Google Chrome と Chrome Driver のセットアップで苦労したので共有します…

## Dockerfile の例

1. wget などのインストールコマンドのインストール
2. Google Chrome のインストール
3. Chrome Driver のインストール
4. Selenium を pip でインストール

```dockerfile
FROM python:3.12-slim
WORKDIR /app/
COPY . /app/
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    unzip \
    jq \
    gnupg \
    && rm -rf /var/lib/apt/lists/*
# Google Chromeのインストール
RUN wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add - \
    && echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google-chrome.list \
    && apt-get update \
    && apt-get install -y google-chrome-stable \
    && rm -rf /var/lib/apt/lists/*
# ChromeDriver(ver115以降)の最新のStableバージョンをダウンロードし、/app/code/にインストールする
RUN LATEST_VERSION=$(wget -qO- https://googlechromelabs.github.io/chrome-for-testing/last-known-good-versions-with-downloads.json | jq -r '.channels.Stable.version') \
    && DOWNLOAD_URL=$(wget -qO- https://googlechromelabs.github.io/chrome-for-testing/last-known-good-versions-with-downloads.json | jq -r '.channels.Stable.downloads.chromedriver[] | select(.platform == "linux64").url') \
    && wget -O /tmp/chromedriver.zip $DOWNLOAD_URL \
    && unzip /tmp/chromedriver.zip -d /app/code/ \
    && rm /tmp/chromedriver.zip \
    && mv /app/code/chromedriver-linux64/chromedriver /app/code/chromedriver \
    && rm -rf /app/code/chromedriver-linux64 \
    && chmod +x /app/code/chromedriver
# Seleniumのインストール
RUN pip3 install -r requirements.txt
RUN useradd -m streamlit
WORKDIR /app/code/
ENV AWS_DEFAULT_REGION=us-west-2
ENV PATH="/app/code:${PATH}"
CMD [ "streamlit", "run", "app.py", "--server.port", "8501", "--server.address", "0.0.0.0" ]

# カレントディレクトリにChromeDriverをインストールする場合の例
# LATEST_VERSION=$(wget -qO- https://googlechromelabs.github.io/chrome-for-testing/last-known-good-versions-with-downloads.json | jq -r '.channels.Stable.version') \
#     && DOWNLOAD_URL=$(wget -qO- https://googlechromelabs.github.io/chrome-for-testing/last-known-good-versions-with-downloads.json | jq -r '.channels.Stable.downloads.chromedriver[] | select(.platform == "linux64").url') \
#     && wget -O ./chromedriver.zip $DOWNLOAD_URL \
#     && unzip ./chromedriver.zip -d ./ \
#     && rm ./chromedriver.zip \
#     && mv ./chromedriver-linux64/chromedriver ./chromedriver \
#     && rm -rf ./chromedriver-linux64 \
#     && chmod +x ./chromedriver
```

## Python Selenium コード例

`service = Service("/app/code/chromedriver")`で Chrome Driver のパス設定をして実行すること

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service

chrome_options = Options()
chrome_options.add_argument("--headless")
chrome_options.add_argument("--disable-gpu")
chrome_options.add_argument("--no-sandbox")
chrome_options.add_argument("--window-size=1920,1080")
chrome_options.add_argument("--disable-web-security")
chrome_options.add_argument("--disable-dev-shm-usage")
chrome_options.add_argument("user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36")
service = Service("/app/code/chromedriver") # Chrome Driverをインストールしたパスに合わせること!!
driver = webdriver.Chrome(service=service, options=chrome_options)
driver.get(url)
```

## 実行例

![](/images/20240821_docker-python-selenium/selenium.gif)

## その他コード

https://github.com/Flupinochan/MyBlog/tree/master/CDK/docker/streamlit

## おまけ

Claude に煽られてる気がする…
(まあ事実だからいいか w)
![](/images/20240821_docker-python-selenium/selenium2.png =500x)
