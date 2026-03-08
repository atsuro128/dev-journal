# 開発コマンド

## 起動
```
docker compose up -d
```

## Backend
```
cargo build
cargo test
cargo clippy
cargo fmt --check
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
sqlx migrate run
```
