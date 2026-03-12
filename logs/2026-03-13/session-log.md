## 00:23 セッション
- 作業: Dev Container 内で pip install が失敗する原因を調査（ファイアウォールのホワイトリストに pypi.org が未登録）
- 作業: コミット 8a00044（firewall-local-domains.txt による拡張ポイント）を取り消し、ドメインを init-firewall.sh のホワイトリストに直接統合
- 判断: firewall-local-domains.txt の仕組みを廃止し、init-firewall.sh に直接記載する方式に変更（理由: Dockerfile が init-firewall.sh のみを /usr/local/bin/ にコピーするため、別ファイル参照がコンテナ内で機能しない構造的問題があった）
