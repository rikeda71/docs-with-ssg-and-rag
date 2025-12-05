# docs-with-ssg-and-rag
SSG と Cloudflare で作るドキュメントシステム

## Usage

1. https://developers.cloudflare.com/fundamentals/api/get-started/create-token/ を参考に、 cloudflare pages にデプロイするための api token を発行
2. https://docs.github.com/ja/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets を参考に、1 の値を CLOUDFLARE_API_TOKEN という名称で登録
    - [deploy.yaml](.github/workflows/deploy.yaml) で利用
