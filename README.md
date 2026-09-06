# LangGraphを利用したAIエージェント

LangGraphの`State`・`Node`・`Edge`・`Conditional Edge`を利用し、
ユーザーの質問に応じてRAGまたはWeb検索を選択して回答するAIエージェントを構築しました。

これまでに作成したRAGとWeb検索の処理をLangGraph上で統合し、
回答生成後にReflectionを行う構成としています。

## 主な機能

- LangGraphによるワークフロー制御
- Stateによる状態管理
- Route NodeによるRAG / Web検索の選択
- Conditional Edgeによる条件分岐
- 社内文書を想定したRAG
- Tavilyを利用したWeb検索
- Qwen3-4BによるローカルLLM実行
- Reflectionによる回答評価
- retry時の再生成
- RAG回答時の参照元表示
- 回答不能時の定型応答

## 全体構成

    START
      ↓
    route
      ↓
    Conditional Edge
     ↙        ↘
    RAG       Web
     ↘        ↙
      answer
        ↓
    reflection
        ↓
    Conditional Edge
     ↙        ↘
    retry      END
      ↓
    route

## 使用技術

- Python
- LangGraph
- LangChain
- Qwen3-4B
- Hugging Face Transformers
- bitsandbytes
- Chroma
- multilingual-e5-base
- Tavily
- Jupyter Notebook

## RAGについて

`RAG_test_data`には、社内文書を想定したダミーデータを配置しています。

主なカテゴリは以下です。

- 人事
- 労務
- 総務
- 情報システム

RAGで回答した場合は、回答とともに参照したファイル名と保存場所を表示します。

`create_sample_data.ipynb`を実行することで、RAG用のダミーデータを再生成できます。

## Router設計

質問内容に応じて、最初にRAGまたはWeb検索のどちらを使用するか判定します。

社内制度、勤怠、福利厚生、社用PCなどの質問はRAGへ振り分けます。

一方、最新ニュース、外部サービス、技術動向など、
外部の最新情報が必要な質問はWeb検索へ振り分けます。

### RAGからWeb検索へ自動Fallbackしない理由

社内情報としてRAGへ振り分けた質問について、
RAGに回答根拠が存在しない場合でも自動的にWeb検索へ切り替えない設計としています。

これは、Web上の一般的な情報を会社固有の制度として回答してしまうことを防ぐためです。

参照文書に回答根拠がない場合は、

> 参照した情報には、質問に該当する情報がありません。

と回答して処理を終了します。

## Reflection

回答生成後、Reflection Nodeで回答内容を評価します。

以下の場合は`retry`と判定します。

- 参考情報にない内容が回答に含まれている
- 質問と関係のない内容が含まれている
- 回答内容に矛盾がある

問題がなければ`ok`として終了します。

`retry`の場合はRoute Nodeへ戻り、再度処理を行います。

無限ループを避けるため、再生成回数には上限を設定しています。

また、Answer Nodeで「参照情報から回答できない」と判定した場合は、
回答不能ケースとしてReflectionをスキップし、`ok`として終了します。

## ファイル構成

    langgraph-ai-agent/
    ├─ 03_LangGraphを利用したAIエージェント.ipynb
    ├─ create_sample_data.ipynb
    ├─ RAG_test_data/
    │  ├─ 人事/
    │  ├─ 労務/
    │  ├─ 総務/
    │  └─ 情報システム/
    └─ README.md

## ダミーデータについて

RAGで使用するダミー社内文書は、これまでのPortfolioで作成したデータを再利用しています。

本リポジトリ単体でも内容を確認できるよう、
`RAG_test_data`フォルダをリポジトリ内に配置しています。

Notebookからは相対パスで読み込みます。

    folder_path = Path("RAG_test_data")

そのため、特定のローカル環境に依存する絶対パスは使用していません。

## 実行の流れ

1. 必要なライブラリをインストールします。
2. Tavily API Keyを環境変数へ設定します。
3. `RAG_test_data`をリポジトリ直下に配置します。
4. `03_LangGraphを利用したAIエージェント.ipynb`を上から順に実行します。
5. 最後のセルで質問を入力します。

質問内容に応じて、RAGまたはWeb検索が自動的に選択されます。

## 出力例

RAGを使用した場合は、回答とあわせて参照元を表示します。

    【Route】
    rag

    【回答】

    有給休暇の申請方法は以下の通りです。
    ...

    【出典】
    ファイル名: 有給休暇申請書.txt
    保存場所: 人事\申請\有給休暇申請書.txt

    【Reflection】
    ok

    【Iteration】
    0

参照文書に回答根拠が存在しない場合は、

    参照した情報には、質問に該当する情報がありません。

と回答します。

## 今後の改善点

- `has_answer`は現在、生成された回答文から判定しているため、回答可能性そのものを誤判定する可能性があります。
- Reflectionの評価理由を次回の回答生成へ直接渡していないため、retry時に同種の回答が再生成される場合があります。
- Web検索結果の鮮度や本文抽出品質によって回答精度が左右されます。
- Web検索結果の関連性や取得期間をより厳密に制御する余地があります。
- 社内固有情報と一般情報をさらに分類し、一般情報として回答可能な場合のみRAGからWeb検索へFallbackする構成も検討できます。

## 補足

本Portfolioでは、RAGやWeb検索そのものを新規に作ることよりも、
それらをLangGraph上でNodeとして接続し、
StateとConditional Edgeによってワークフロー全体を制御することを主な目的としています。

RAG、Web検索、回答生成、Reflectionをそれぞれ独立したNodeとして扱うことで、
処理の役割を分離しながら、条件分岐と再実行を含むAIエージェントの構成を実装しました。

## 動作確認環境

- CPU: AMD Ryzen 7 5700X
- RAM: 32GB
- GPU: NVIDIA GeForce RTX 4070 12GB
