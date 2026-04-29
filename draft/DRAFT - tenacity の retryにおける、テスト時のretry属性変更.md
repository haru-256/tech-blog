
- テスト時だけ高階関数の引数を変える
- 伝えたいこと
    - テスト時だけ高階関数の引数を変えたいことがある
    - テスト時だけ高階関数の引数を変える方法について
        - 一般的な方法
        - tenacityのretryの場合
- テストのときだけ、tenacityのretryのmax retry countを変更したい。実行時に高階関数の引数を変えたい。
- 以下のようにpatchを当てることで高階関数の引数を変更できる

```python
def test():
    # Override the retry condition temporarily for this test
    # tenacity evaluates decorators at import time, so we must mock the retry object itself
    mocker.patch.object(post_with_retry.retry, "stop", stop_after_attempt(2))  # type: ignore

    # Expect HTTPStatusError after retries exhausted (from log_and_raise_final_error)
    with pytest.raises(httpx.HTTPStatusError):
        await post_with_retry(mock_client, "http://test.com", {}, {})

    # Should stop after 2 attempts
    assert mock_client.post.call_count == 2

@retry(
    stop=stop_after_attempt(config.max_retry_count),
    wait=wait_retry_after,
    retry=retry_if_result(is_rate_limit),
    before=before_log,
    retry_error_callback=log_and_raise_final_error,
)
async def post_with_retry(
    client: httpx.AsyncClient,
    url: str,
    params: dict[str, Any],
    json: dict[str, Any],
    headers: dict[str, str] | None = None,
) -> httpx.Response: ...
```

- その理由

`tenacity` ライブラリの作りによるものです。

Pythonのデコレータは、関数を受け取って「別の関数（ラッパー）」を返す仕組みになっていますが、`tenacity` の `@retry` デコレータは単なる関数ではなく、**独自の設定オブジェクトへの参照を持った特殊なラッパーオブジェクト**（正確には `Retrying` クラスの動作を内包したラッパー）を返します。

`tenacity` は親切設計になっていて、デコレートされた元の関数に対して、retry という属性**（プロパティ）を自動的にくっつけてくれます。

この `post_with_retry.retry` にアクセスすると、デコレータとして渡された設定（`stop`, retry など）を管理している `AsyncRetrying` (非同期関数の場合) というクラスのインスタンスがそのまま取得できます。
なぜこれが便利なのか？

通常、デコレータに渡した引数は後から変更できません。しかし `tenacity` はこの `.retry` 属性を公開してくれているおかげで、以下のような**「実行時の一時的な設定変更（今回のようなテスト時のモックなど）」が簡単に行える**ようになっています。

さらに、テストだけではなく本番コードの中でも以下のように設定を変えたり統計を取ったりすることができます：

```python
# 現在のリトライ設定を見る
print(post_with_retry.retry.stop)

# これまでのリトライ実行に関する統計情報を取得する（成功回数や失敗回数など）
print(post_with_retry.retry.statistics)

```

そのため、テストコードではこの性質を利用して、`post_with_retry.retry.stop` 属性だけを「2回でストップする条件（`stop_after_attempt(2)`）」に書き換えることができました。静的型チェッカー（mypy）は「通常の関数には `.retry` なんて属性はないはずだ」と怒るため `# type: ignore` が必要になりますが、実行時には確実に存在して機能します。