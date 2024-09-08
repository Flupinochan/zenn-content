---
title: "LlamaとGitHubで実現するリポジトリQ&A機能"
emoji: "📘"
type: "tech"
topics: ["LlamaIndex", "OpenAI", "Streamlit", "faiss", "GitHub"]
published: true
---

Amazon CodeCatalyst や Cursor のようなリポジトリのコードに対して Q&A する仕組みを作ってみた

## 前提

最低限、以下は必要(pipreqs より取得)

```bash
pip install openai
pip install streamlit
pip install faiss-cpu
pip install llama-index
pip install llama-index-readers-github
```

## コード例

・LLM: OpenAI
・Vector Store: FAISS
・ChatBot: Streamlit
・Repository: GitHub

```python
import streamlit as st
import openai
import faiss
from llama_index.core import load_index_from_storage, VectorStoreIndex, StorageContext, Settings
from llama_index.llms.openai import OpenAI
from llama_index.readers.github import GithubRepositoryReader, GithubClient
from llama_index.vector_stores.faiss import FaissVectorStore

# OpenAI設定
openai.api_key = "" # APIキー
Settings.llm = OpenAI(
    model="gpt-4o-mini",
    temperature=0.1,
    max_tokens=16000,
)
# Vector Store設定
d = 1536
faiss_index = faiss.IndexFlatL2(d)
# GitHubクライアントの設定
github_token = "" # トークン
github_client = GithubClient(github_token=github_token, verbose=True)
# タイトル
st.title("GitHub Q&A with Llama")
# GitHubリポジトリ情報入力用のフォーム
with st.form(key="repo_form"):
    owner = st.text_input("Owner", value="Flupinochan")
    repo = st.text_input("Repository", value="zenn-content")
    branch = st.text_input("Branch", value="master")
    filter_directories_input = st.text_input("EXCLUDE Filter Directories (comma-separated)", value="books,images")
    filter_file_extensions_input = st.text_input("INCLUDE Filter File Extensions (comma-separated)", value=".md,.ts,.tsx,.py")
    submit_button = st.form_submit_button("Submit")
# セッション初期化
if "query_engine" not in st.session_state:
    st.session_state.query_engine = None
if "chat_history" not in st.session_state:
    st.session_state.chat_history = []
# Submitボタン実行時の処理
if submit_button:
    with st.spinner("Initializing index..."):
        try:
            filter_directories = [dir.strip() for dir in filter_directories_input.split(",") if dir.strip()]
            filter_file_extensions = [ext.strip() for ext in filter_file_extensions_input.split(",") if ext.strip()]
            reader = GithubRepositoryReader(
                github_client=github_client,
                owner=owner,
                repo=repo,
                use_parser=True,
                verbose=False,
                filter_directories=(filter_directories, GithubRepositoryReader.FilterType.EXCLUDE),
                filter_file_extensions=(filter_file_extensions, GithubRepositoryReader.FilterType.INCLUDE),
                concurrent_requests=5,
                retries=5,
                timeout=300,
            )
            documents = reader.load_data(branch=branch) # GitHubリポジトリを読み込む
            vector_store = FaissVectorStore(faiss_index=faiss_index)
            storage_context = StorageContext.from_defaults(vector_store=vector_store)
            index = VectorStoreIndex.from_documents(documents, storage_context=storage_context) # Vectorに保存/Index作成
            index.storage_context.persist()
            vector_store = FaissVectorStore.from_persist_dir("./storage")
            storage_context = StorageContext.from_defaults(vector_store=vector_store, persist_dir="./storage") # カレントディレクトリstorageに永久保存
            st.session_state.query_engine = load_index_from_storage(storage_context=storage_context).as_query_engine(streaming=True, similarity_top_k=1)
            st.success("Index initialized successfully.")
        except Exception as e:
            st.error(f"An error occurred: {e}")

# チャットメッセージの入力欄と送信ボタン
user_message = st.text_input("Enter your message:")
if st.button("Send"):
    if user_message:
        if st.session_state.query_engine:
            with st.spinner("Processing your message..."):
                try:
                    instruction = "以下の内容について、長文で良いので詳細かつ丁寧に最低でも2000字以上でMarkdown形式で回答してください。"
                    final_prompt = f"{instruction}\n{user_message}"
                    streaming_response = st.session_state.query_engine.query(final_prompt) # 質問実行
                    placeholder = st.empty()
                    chat_history = st.session_state.get("chat_history", [])
                    # ストリーミング出力の処理
                    full_text = ""
                    for text in streaming_response.response_gen:
                        full_text += text
                        placeholder.write(full_text)
                    chat_history.append(full_text)
                    st.session_state.chat_history = chat_history
                except Exception as e:
                    st.error(f"An error occurred while querying: {e}")
        else:
            st.write("Please submit the form to initialize the index.")
    else:
        st.write("Please enter a message.")
```

## 実行例

指定した GitHub リポジトリを読み込む
![](/images/20240901_llama-github-chat/llama-github-chat1.gif)
質問する
![](/images/20240901_llama-github-chat/llama-github-chat2.gif)
<br>
<br>
P.S. GitHub REST API で一覧取得する方法と大差なかった…
