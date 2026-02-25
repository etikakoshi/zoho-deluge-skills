# Zoho Deluge 実装スキル

Zoho CRM の Deluge 関数を実装・レビューする際の言語制約、禁止API、invokeurl/COQL ルール、関数テンプレート、ログ規則をまとめたスキル。`.deluge` ファイル編集時や「Deluge で〇〇して」「Zoho CRM の関数を書いて」といったリクエスト時に参照する。

---

## 1. 言語制約・禁止事項

実装時は以下を厳守すること。

### 禁止構文

- **C言語スタイルの for**: 使用禁止。`for each item in list` のみ可。
- **while ループ**: 使用禁止。ループは「回数用リスト」＋ `for each` で代替。
- **三項演算子 (?:)**: 使用禁止。`?` を記述すると "Unexpected token found '?'" になる。if 文で代替すること。ifnull は可。
- **関数のネスト**: 1関数1タスク。関数内で別関数を定義しない。
- **invokeurl ブロック内のコメント**: 禁止（`//` も `/* */` も不可）。
- **行末コメント**: 禁止。コメントは独立行に書く。
- **1行 if 内での代入**: 禁止。必ずブロック `{ }` で囲む。

### 禁止API・メソッド

- **使用禁止**: `getType()`, `instanceof`, `isList()`, `join()`, `repeat()`, `keySet()`, `compareTo()`
- **toint()**: 存在しない。整数変換は `toLong()` を使用。
- **存在しない組み込み関数**: 例 `getMonthStart()`, `getMonthEnd()` は使用しない。日付は `addDay()`, `addMonth()` 等で実装。

### その他禁止・注意

- **sendmail の from**: 必ずハードコードされた文字列リテラル。
- **zoho.crm.attachFile**: 4引数呼び出し・File 以外の渡し方は禁止。
- **組織変数**: `zoho.crm.getOrgVariable()` を使用。`organization.get()` 等の誤った取得方法は使用しない。

### 関数種別と戻り値

- **automation**: 必ず `void`。`return;` のみ。throw は使用不可。
- **button（カスタムボタン）**: `String button.関数名(...)`。全パスで return 必須。try-catch は避け、単純な if で分岐する形式を推奨。
- **standalone（単独関数）**: 戻り値は **string 型のみ**使用可能。bool / int 等を返すと "Invalid return Type" エラーになる。真偽や数値で返したい場合は `"true"` / `"false"` や数値の文字列を返し、呼び出し側で `ret == "true"` のように比較すること。
- **入力規則**: `map validation_rule.関数名(string crmAPIRequest)`。response に `status` と `message` を設定して return。

### 型・代替パターン

- リスト結合: `join()` 禁止 → 空文字＋`for each` で連結。
- ループ回数指定: 長さ n の文字列を `"".leftPad(n)` で作り、`replaceAll`/`toList()` 等でリスト化して `for each`。
- リスト型判定: `isList()` 禁止 → `reason.toString()` の `startsWith("[")` かつ `endsWith("]")` で判定。

---

## 2. API・テンプレート・ログ

### 組織変数

- `dc` = "jp", `version` = "v8"
- 取得: `zoho.crm.getOrgVariable("dc")`, `zoho.crm.getOrgVariable("version")`
- 追加の組織変数は厳禁（定義済みのみ使用）。

### ベースURL構築

- Sandbox: `baseDomain = "sandbox.zohoapis." + dc`
- 本番: `baseDomain = "www.zohoapis." + dc`
- API ベース: `url_base = "https://" + baseDomain + "/crm/" + version + "/"`

### invokeurl ルール

- パラメータ: 外側 Map → 送信前に `.toString()` を必ず実行。
- invokeurl ブロック内にコメントを書かない。
- レスポンスは**全文**をログ出力（ステータスコードのみにしない）。

### COQL

- **IN 演算子**: サポートされていない。複数値は OR 条件を動的構築。
- **サブフォーム**: COQL ではサブフォームのフィールドを直接 SELECT 不可。親 ID 取得 → 個別 GET で詳細取得。
- 括弧: 複数条件は階層的に括弧でグループ化。文字列はシングルクォート、数値はクォートなし。
- エンドポイント: `https://` + baseDomain + `/crm/` + version + `/coql`。payload は Map（List ではない）。

### 関数テンプレート（順序厳守）

1. 関数宣言  
2. 開き `{`（同一行）  
3. ブロックコメント（function_name, label_name, motivation, processing_flow, related_functions）  
4. Config / 変数初期化  
5. データ準備  
6. API 呼び出し（invokeurl、ブロック内コメントなし）  
7. catch（任意）  
8. 閉じ `}`  

### 関数呼び出し（&arguments）

- URL: `baseUrl + '&arguments=' + jsonString`
- JSON は手組みの**有効な JSON 文字列**。Map.toString() ではなく文字列連結で構築。
- POST で送信。parameters フィールドは使わない。
- JSON のキーは引数名と完全一致（大文字小文字区別）。

### ログ

- 日本語で記述。
- 常に API の**完全なレスポンス**を出力。

### ドキュメント優先

- Zoho CRM / Deluge の API・メソッドは公式ドキュメントを最優先。パラメータ名・接続名・必須ヘッダは公式例に厳密に従う。
- 参照コードを指定された場合は、その実装とスタイルを最優先で模倣する。

---

## 3. 実際の使用例（実装時はこの形に従う）

以下は動作確認されているパターン。実装・レビュー時はこれらの形を参照すること。

### 3.1 組織変数と invokeurl の基本形

```deluge
dc = zoho.crm.getOrgVariable("dc");
version = zoho.crm.getOrgVariable("version");
isSandbox = false;
if (isSandbox)
{
    baseDomain = "sandbox.zohoapis." + dc;
}
else
{
    baseDomain = "www.zohoapis." + dc;
}
url_base = "https://" + baseDomain + "/crm/" + version + "/";

params = Map();
params.put("data", dataList);

response = invokeurl
[
    url       : url_base + "Leads"
    type      : POST
    parameters: params.toString()
    connection: "crm_module"
];
info "APIレスポンス全文: " + response;
```

### 3.2 automation 関数のテンプレート（順序厳守）

```deluge
void automation.ExampleFunction()
{
    /*
    function_name   : ExampleFunction
    label_name      : サンプル処理
    motivation      : 〇〇の動作確認
    processing_flow :
      - 設定値を config Map に定義
      - データ準備
      - API 呼び出しとログ出力
    */

    dc = zoho.crm.getOrgVariable("dc");
    version = zoho.crm.getOrgVariable("version");
    baseDomain = "www.zohoapis." + dc;
    url_base = "https://" + baseDomain + "/crm/" + version + "/";

    params = Map();
    params.put("key", "value");

    try
    {
        resp = invokeurl
        [
            url       : url_base + "Module"
            type      : POST
            parameters: params.toString()
            connection: "crm_module"
        ];
        info "処理結果: " + resp;
    }
    catch (e)
    {
        info "エラー: " + e;
    }
}
```

### 3.3 カスタムボタン関数（全パスで return、try-catch は避ける）

```deluge
String button.myButton(String recordId)
{
    if (recordId == null || recordId == "")
    {
        return "エラー: レコードIDが指定されていません";
    }
    if (条件A)
    {
        return "成功メッセージ";
    }
    return "未対応ケース";
}
```

### 3.4 リストの結合（join 禁止の代替）

```deluge
combined = "";
for each item in list
{
    if (combined != "")
    {
        combined = combined + ", ";
    }
    combined = combined + item;
}
```

### 3.5 if での代入は必ずブロックで（1行代入禁止）

```deluge
if (pos1 == -1)
{
    pos1 = 1000;
}
if (value == null)
{
    value = "";
}
```

### 3.6 COQL の OR 条件（IN は使わない）

```deluge
if (unique_phones.size() == 1)
{
    query = query + "(field14 = '" + unique_phones.get(0) + "')";
}
else if (unique_phones.size() == 2)
{
    query = query + "((field14 = '" + unique_phones.get(0) + "') OR (field14 = '" + unique_phones.get(1) + "'))";
}
else if (unique_phones.size() > 2)
{
    query = query + "(((field14 = '" + unique_phones.get(0) + "') OR (field14 = '" + unique_phones.get(1) + "'))";
    for each phone in unique_phones.subList(2, unique_phones.size())
    {
        query = query + " OR (field14 = '" + phone + "')";
    }
    query = query + ")";
}
```

### 3.7 関数呼び出し（&arguments）の形

```deluge
baseUrl = "https://www.zohoapis.com/crm/" + version +
          "/functions/MyFunction/actions/execute?auth_type=apikey&zapikey=<ZAPIKEY>";
jsonArgs = "{\"dealId\":\"" + dealId + "\",\"name\":\"" + name + "\"}";
execUrl = baseUrl + "&arguments=" + jsonArgs;
resp = invokeurl
[
    url  : execUrl
    type : POST
];
info resp;
```

### 3.8 invokeUrl 直後の API エラー判定（4xx でも例外が出ないため必須）

```deluge
response = invokeurl
[
    url       : url_base + "coql"
    type      : POST
    parameters: coqlMap.toString()
    connection: "crm_module"
];
info "COQL レスポンス全文: " + response;

if (response.get("status") != null && response.get("status").toString() == "error")
{
    apiCode = "";
    if (response.get("code") != null) { apiCode = response.get("code").toString(); }
    apiMsg = "";
    if (response.get("message") != null) { apiMsg = response.get("message").toString(); }
    result.put("error", "COQL エラー: " + apiCode + " - " + apiMsg);
    return result.toString();
}

data = List();
if (response.get("data") != null)
{
    data = response.get("data");
}
```

より多くの実装例（ループ代替、attachFile、クライアントスクリプト等）は、プロジェクト内に `.claude/rules/core/zoho-deluge-rules.md` がある場合はそちらを、または本スキル同梱の `references/usage-examples.md` を参照すること。

---

## 4. 参照の優先順位（プロジェクト内）

1. 本スキル（上記要約）
2. 完全版ルール（プロジェクトに `.claude/rules/core/zoho-deluge-rules.md` がある場合はそちらにサンプルコード・詳細あり）
3. 公式ドキュメント（Zoho Deluge / CRM API）
4. プロジェクト仕様書（各プロジェクトの関数仕様書。03 インデックスはプロジェクト別に管理）

---

*このスキルは GitHub 等で共有する共通ルール用。プロジェクト別の仕様書インデックス（03）は各リポジトリの `skills/03_zoho-deluge-project-specs-index.md` 等で管理すること。実装時は「3. 実際の使用例」のコード形を参照し、参照コードを指定された場合はそれを最優先で模倣すること。*
