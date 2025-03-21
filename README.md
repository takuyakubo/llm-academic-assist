# LLM学術文書支援システム

LLM学術文書支援システムは、大規模言語モデル(LLM)を活用して数学、物理学、コンピュータサイエンスなどの学術文書の理解と学習を支援するためのプラットフォームです。

## 主な機能

- プロジェクト単位での文書管理
- PDF・画像からのテキスト・数式・図表の抽出と構造化
- LLMを活用した文書内容に関する質問応答
- 文書の自動要約と注釈付け
- 学習ノートの自動生成
- プロジェクト間の知識共有と相互参照

## システムアーキテクチャ

- **バックエンド**: Python 3.10+, FastAPI
- **フロントエンド**: Next.js 14+, React 18+, TypeScript
- **データベース**: MongoDB 6.0+
- **ストレージ**: AWS S3 または同等のオブジェクトストレージ
- **コンテナ化**: Docker, Docker Compose
- **CI/CD**: GitHub Actions
- **デプロイ**: AWS ECS/EKS または Vercel + AWS Lambda
- **LLM API**: Claude API, OpenAI API (GPT-4)
- **文書処理**: PyPDF2, PyMuPDF, Tesseract OCR, MathJax, KaTeX

## ドキュメント

詳細な設計や開発ドキュメントは [`/documents`](./documents/) ディレクトリに格納されています。
