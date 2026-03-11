# 開発コマンド

## 起動
```
docker compose up -d
```

## Backend
```
go build ./...
go test ./...
go vet ./...
golangci-lint run
gofmt -l .
```

## Frontend
```
npm run dev
npm test
npm run lint
npm run build
```

## マイグレーション
```
# マイグレーションツールは Step 3 で選定（golang-migrate / goose 等）
```
