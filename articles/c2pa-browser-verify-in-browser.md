---
title: "C2PA 署名をブラウザだけで暗号検証する — ファイルを1バイトも送らずに"
emoji: "🔏"
type: "tech"
topics: ["c2pa", "webassembly", "javascript", "セキュリティ", "生成ai"]
published: true
---


画像に「誰が・何で作ったか」の署名を埋め込む **C2PA（Content Credentials）** は、Adobe やソニー、ニコン、Google などが採用を進めています。日本でも、新聞社や自治体が発信者情報を付ける **Originator Profile（OP）** の実装が進んでいます。

ただ、署名を **付ける側** の話に比べて、**受け取った側** が手元で検証する話はあまり見かけません。

この記事では、受け取った画像の C2PA 署名を **ブラウザの中だけで暗号検証する** 方法を書きます。サーバーにファイルを送らず、ページ自体も外部と通信できない状態で検証します。公式の公開テストファイルで試した結果と、つまずいた点もまとめました。

## この記事で分かること

- 公式 SDK `@contentauth/c2pa-web` を、ビルド工程なしの静的サイトに同梱する方法
- 「外部に通信しない」を CSP で強制したまま、wasm と Worker を動かす設定
- 検証結果の「無効」を、改ざん・証明書の期限切れ・信頼リスト未照合に分けて扱う方法
- 公開テストファイル 8 点での結果と、期待どおりにならなかった 1 件

## なぜブラウザだけでやるのか

検証したい画像は、保険の請求写真や本人確認の書類、取材で受け取った素材など、外に出したくないものであることが多いです。「検証サービスにアップロードしてください」では使えません。

そこで、次の 3 つを満たすことを目標にしました。

1. 解析は File API と wasm で完結させ、ファイルを送信しない
2. それを **ページ側の約束ではなく CSP で強制する**（`connect-src 'self'`）
3. オフラインでも動く（CDN に依存しない）

## 1. SDK を静的サイトに同梱する

C2PA の公式 JavaScript SDK は `@contentauth/c2pa-web` です。中身は Rust 製の `c2pa-rs` を wasm にしたもので、検証は Web Worker 内で行われます。

```bash
npm i -D @contentauth/c2pa-web@0.15.3 esbuild
```

そのまま `<script type="module">` で読もうとすると失敗します。内部で `highgain` などの依存を **素のモジュール名** で import しているためです。そこで esbuild で 1 ファイルにまとめ、wasm と一緒に `public/vendor/` に書き出します。

```js
// scripts/vendor.mjs
import { build } from 'esbuild';
import { copyFile, mkdir } from 'node:fs/promises';

const pkg = 'node_modules/@contentauth/c2pa-web/';
await mkdir('public/vendor/c2pa/', { recursive: true });
await build({
  entryPoints: [`${pkg}dist/index.js`],
  bundle: true, format: 'esm', minify: true, legalComments: 'eof',
  outfile: 'public/vendor/c2pa/c2pa-web.js',
});
await copyFile(`${pkg}dist/resources/c2pa_bg.wasm`, 'public/vendor/c2pa/c2pa_bg.wasm');
```

生成物は JS が約 40 KB、**wasm が約 8.6 MiB** です。生成物はリポジトリに含めておき、配信時にはビルドしません。

## 2. 必要なときだけ読み込む

8.6 MiB をすべての訪問者に読ませるのは重すぎます。C2PA のマニフェストは、JPEG なら APP11 セグメントの JUMBF に入っていて、`c2pa.actions` や `claim_generator` といった文字列を含みます。

そこで、ファイルの **メタデータ領域だけ** を先に走査し、こうした文字列が見つかったときだけ SDK を読み込むようにしました。画素データまで含めると、圧縮データ中の偶然のバイト列に一致することがあるので、走査対象から外しています。

```js
let sdkPromise = null;

async function loadSdk() {
  if (!sdkPromise) {
    sdkPromise = (async () => {
      const mod = await import('../vendor/c2pa/c2pa-web.js');
      const wasmSrc = new URL('../vendor/c2pa/c2pa_bg.wasm', import.meta.url).href;
      const c2pa = await mod.createC2pa({ wasmSrc });
      // 外部への取得はしない（リモートマニフェスト・OCSP）。信頼リストも読み込まない
      const context = new mod.Context({
        verify: { verifyTrust: false, remoteManifestFetch: false, ocspFetch: false },
      });
      return { mod, c2pa, context };
    })().catch(e => { sdkPromise = null; throw e; });
  }
  return sdkPromise;
}

export async function readC2pa(file) {
  const { mod, c2pa, context } = await loadSdk();
  const reader = await mod.Reader.fromBlob(c2pa, file.type || undefined, file, context);
  if (!reader) return null;           // マニフェスト無し
  try {
    return await reader.manifestStore();
  } finally {
    await reader.free();               // wasm 側のメモリを解放する
  }
}
```

`Reader.fromBlob` の第 2 引数（形式）は省略もできます。ただし公式 README は、省略するとバイト列から推測するので遅く、不安定になりうると注意しています。ブラウザの `file.type` が空のこともあるため、コンテナの先頭バイトから判定した形式を渡すのが安全です。

## 3. CSP で「外部と通信できない」を強制する

ページから外部に通信できないことを、ブラウザに強制させます。配信時のヘッダーはこうしました（Cloudflare の静的アセットや Netlify では `_headers` ファイルで指定できます）。

```text
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'wasm-unsafe-eval';
  worker-src 'self' blob:;
  connect-src 'self';
  img-src 'self' data: blob:;
  media-src blob:;
  style-src 'self' 'unsafe-inline';
  form-action 'none'; frame-ancestors 'none'; base-uri 'none'; object-src 'none'
```

wasm を動かすために、必須の項目が 2 つあります。

- **`'wasm-unsafe-eval'`**: これが無いと wasm をコンパイルできません。`'unsafe-eval'` を丸ごと許可する必要はありません。
- **`worker-src blob:`**: c2pa-web は検証用の Worker を **blob URL で起動** します。`createC2pa` に `workerSrc` を渡すと別ファイルの Worker も使えますが、その場合は https の URL しか受け付けません。同梱するだけなら `blob:` を許可するのが簡単です。

`connect-src 'self'` のままでも、同じオリジンの wasm は取得できます。SDK は wasm を integrity（ハッシュ）付きで取得していて、中身が差し替えられていれば読み込みに失敗します。

:::message
公開してから気づいたのですが、Cloudflare で新しいドメインを追加すると **Web Analytics（RUM）の自動挿入** が有効になっていることがあります。CSP が `static.cloudflareinsights.com` をブロックしてくれていたので計測データは送られていませんでしたが、「アクセス解析なし」とうたうなら、ダッシュボードで Disable にしておくべきです。
:::

## 4. 「無効」を 3 種類に分ける

`manifestStore()` は `validation_state`（`Valid` / `Invalid` / `Trusted`）と、失敗のコード（`validation_status` / `validation_results.activeManifest.failure`）を返します。

最初は「Invalid なら改ざん」と扱っていましたが、公開テストファイルで試すと、それでは誤解を招くことが分かりました。現在は、コードを次の 3 種類に分けています。

| 種類 | 例 | 意味 | 表示 |
|---|---|---|---|
| 内容・署名の不一致 | `claimSignature.mismatch` `assertion.dataHash.mismatch` `assertion.hashedURI.mismatch` | 署名後にファイルや記録が変わった | 改ざんの可能性 |
| 証明書の状態 | `signingCredential.expired` / `revoked` | 内容は壊れていないが、署名者を確認済みとは扱えない | 来歴として使わない |
| 信頼リスト・外部取得 | `signingCredential.untrusted` など | オフラインなので照合していないだけ | 署名は有効（注記つき） |

```js
const NON_INTEGRITY = /untrusted|trust|ocsp|timeStamp\.(untrusted|outsideValidity)|remote|signingCredential\.(untrusted|ocsp)/i;
const CREDENTIAL = /signingCredential\.(expired|revoked|invalid)/i;

// 2 か所に同じコードが入ることがあるので重複を除く
const codes = [...new Set([
  ...(store.validation_status ?? []),
  ...(store.validation_results?.activeManifest?.failure ?? []),
].filter(s => s?.code && s.success !== true).map(s => s.code))];

const integrity = codes.filter(c => !NON_INTEGRITY.test(c) && !CREDENTIAL.test(c));
const credential = codes.filter(c => CREDENTIAL.test(c));
const status = integrity.length ? 'invalid'
  : credential.length ? 'credential'
  : store.validation_state === 'Invalid' ? 'invalid' : 'valid';
```

もう 1 つ決めたのは、**改ざんの可能性を「AI 生成の疑い」とは別の評価軸にする** ことです。署名が壊れていることと、AI で作られたことは別の問いです。両者を 1 つのスコアに混ぜると、AI 生成ではない改ざんサンプルが「AI 生成の疑いが強い」と表示されてしまいました。

## 5. 公式の公開テストファイルで試す

C2PA は、検証器の実装者向けに [公開テストファイル](https://github.com/c2pa-org/public-testfiles) を用意しています（CC BY-SA 4.0）。legacy/1.4 から 8 点を選び、Chrome で検証しました。

| ファイル | 公式の期待 | 結果 |
|---|---|---|
| adobe-20220124-A | C2PA なし | C2PA なし ✓ |
| adobe-20220124-C / CA | 有効 | Valid ✓ |
| adobe-20220124-E-sig-CA | 署名不一致 | `claimSignature.mismatch` ✓ |
| adobe-20220124-E-dat-CA | ハッシュ不一致 | `assertion.dataHash.mismatch` ✓ |
| adobe-20220124-E-uri-CA | アサーション改ざん | `assertion.hashedURI.mismatch` ✓ |
| nikon-20221019-building | 証明書期限切れ + 署名不一致 | `signingCredential.expired` のみ |
| truepic-20230212-camera | エラーなし（公開時の参照結果） | `assertion.dataHash.mismatch` ✗ |

改ざんの 3 種類は、すべて正しいコードで検出できました。

一致しなかったのは 2 件です。

- **Nikon**: 公式の参照結果は期限切れと署名不一致の 2 つですが、現行の SDK は期限切れの 1 つだけを返しました。どちらにしても「無効」なのは同じなので、表示は「証明書が期限切れ」としています。
- **Truepic**: 2023 年の参照結果ではエラーなしですが、現行の SDK はハッシュ不一致と判定しました。SDK の検証が厳しくなったのか、ファイル側の問題なのかは **まだ切り分けられていません**。正規のスマホ撮影画像が「改ざんの可能性」と表示されると影響が大きいので、新しい世代の C2PA 付き実ファイルで追加検証する予定です。

## 6. やっていないこと（限界）

- **信頼リストの照合はしていない**: 署名の数学的な正しさは検証できますが、「その署名者が信頼できる発行元か」は判定していません。C2PA の信頼リスト（PEM）を同梱して `trust.trustAnchors` に渡せば対応できるはずですが、まだ試していません。
- **リモートマニフェストは見ない**: ファイルの外に置かれたマニフェストは、通信しない方針なので取得しません。
- **C2PA が無いことは何の証明にもならない**: SNS で再圧縮されると、署名は簡単に消えます。「署名が無い＝本物」でも「署名が無い＝偽物」でもありません。

## まとめ

- 公式 SDK は esbuild で 1 ファイルにまとめれば、ビルド工程なしの静的サイトに同梱できます。
- CSP で必須なのは `'wasm-unsafe-eval'` と `worker-src blob:` の 2 つです。`connect-src 'self'` のままで動きます。
- 「無効」は、改ざん・証明書の状態・信頼リスト未照合に分けて表示します。
- 公開テストファイルで検証すると、実装の思い込みがよく見つかります。

---

この仕組みを、画像・動画の AI 生成や加工の手がかりをまとめて確認できる無料ツール [AI生成メディア検証](https://ai-media-check.app) に組み込んでいます。ファイルは送信されません。
