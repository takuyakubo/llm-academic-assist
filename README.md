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

### 主要ドキュメント

- [システム概要](./documents/01_概要/01_システム概要.md) - システムの目的と機能概要
- [用語集](./documents/01_概要/02_用語集.md) - システム全体で使用される用語の定義
- [機能要件](./documents/02_要件定義/01_機能要件.md) - 詳細な機能要件一覧
- [全体アーキテクチャ](./documents/03_システム設計/01_アーキテクチャ/01_全体アーキテクチャ.md) - システム設計の全体像
- [技術仕様統一](./documents/03_システム設計/01_アーキテクチャ/03_技術仕様統一.md) - 使用技術の詳細と統一基準
- [データモデル概要](./documents/03_システム設計/04_データモデル/01_データモデル概要.md) - データ構造の設計
- [API仕様](./documents/05_API仕様/03_OpenAPI仕様_統合.yaml) - RESTful APIの詳細仕様
- [トレーサビリティマトリクス](./documents/06_トレーサビリティマトリクス.md) - 機能要件とAPIの対応関係

### ドキュメント間の関連性

- **機能要件** → **API仕様**: 各機能要件(PM-001など)は[トレーサビリティマトリクス](./documents/06_トレーサビリティマトリクス.md)を通じて対応するAPIエンドポイントと紐づけられています
- **API仕様** → **データモデル**: API仕様で使用されるデータ構造は[データモデル概要](./documents/03_システム設計/04_データモデル/01_データモデル概要.md)で定義されています
- **用語統一**: [用語集](./documents/01_概要/02_用語集.md)はシステム全体で一貫した用語使用の基準となります

## 開発ステータス

このリポジトリは現在、設計ドキュメントのみを含んでおり、実装はまだ行われていません。