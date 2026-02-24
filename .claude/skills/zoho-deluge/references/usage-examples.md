# Zoho Deluge 実装例（参照用）

SKILL.md に続く実際の使用例。実装・レビュー時はこの形に従うこと。

---

## ループ回数の指定（while 禁止の代替）

```deluge
iterationCount = 50;
iterationString = "".leftPad(iterationCount);
iterationString = iterationString.replaceAll(" ", ",");
iterationString = iterationString.subString(0, iterationString.length() - 1);
pageIterationList = iterationString.toList();

offset = 0;
page = 0;
for each i in pageIterationList
{
    page = page + 1;
    query = Map();
    query.put("select_query", "select id from Leads limit 200 offset " + offset);
    response = invokeurl
    [
        url       : "https://www.zohoapis.jp/crm/v8/coql"
        type      : POST
        parameters: query.toString()
        connection: "crm_module"
    ];
    if (response == "" || response.get("info").get("more_records") == false)
    {
        break;
    }
    offset = offset + 200;
}
```

---

## zoho.crm.attachFile の正しい使い方（3引数・File オブジェクト）

```deluge
file_data = invokeurl
[
    url : "https://example.com/yourfile.pdf"
    type: GET
];
file_data.setFileName("請求書_20231027.pdf");
attachment_response = zoho.crm.attachFile("Invoices", invoiceId, file_data);
info attachment_response;
```

---

## 標準 API リクエスト（Map → List → Map、.toString() で送信）

```deluge
requestMap = Map();
dataList = List();
dataMap = Map();
dataMap.put("Customer_Name", "山田太郎");
dataMap.put("Customer_ID", "C001");
dataList.add(dataMap);
requestMap.put("data", dataList);

response = invokeurl
[
    url       : "https://www.zohoapis.jp/crm/v8/Customers"
    type      : POST
    parameters: requestMap.toString()
    connection: "crm_module"
];
info "APIレスポンス: " + response;
```

---

## COQL リクエスト

```deluge
queryMap = Map();
queryMap.put("select_query", "select Name, id from AssetAndBookManagement where item_status = '貸出可能'");

response = invokeurl
[
    url       : "https://www.zohoapis.jp/crm/v8/coql"
    type      : POST
    parameters: queryMap.toString()
    connection: "crm_module"
];
info "COQLレスポンス: " + response;
```

---

## ワークフロー無効化して更新（trigger で空を渡す）

```deluge
param = Map();
param.put("data", dataList);
param.put("trigger", {});

upd = invokeurl
[
    url       : "https://www.zohoapis.jp/crm/v8/Sales_Orders"
    type      : POST
    parameters: param.toString()
    connection: "crm_module"
];
info upd;
```

---

## リスト型の判定（isList() 禁止の代替）

```deluge
reasonStr = reason.toString();
if (reasonStr.startsWith("[") && reasonStr.endsWith("]"))
{
}
```

---

## 数値のフォーマット（カンマ区切り）

```deluge
formattedDiscountAmount = totalDiscountAmount.toNumber().toDecimal().toString("##,##0");
```

---

## 文字列から部分取得（indexOf + substring）

```deluge
original_plan = "直葬（事前予約）";
index = original_plan.indexOf("（");
if (index != -1)
{
    plan = original_plan.substring(0, index).trim();
}
```

---

*プロジェクトに `.claude/rules/core/zoho-deluge-rules.md` がある場合は、さらに多くのサンプル（sendmail・クライアントスクリプト・JotForm Webhook 等）をそちらで参照すること。*
