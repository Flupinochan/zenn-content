---
title: "ExcelからCloudFormationテンプレートを生成"
emoji: "👋"
type: "tech"
topics: ["aws", "cloudformation", "python", "jinja"]
published: true
---

## 概要

AWS EC2構築用のExcelパラメータシートをPythonでCloudFormationテンプレートに変換する方法について紹介します

## 使用するPythonライブラリ

- **Jinja**
変数やif、for文を使用して、HTMLやYAMLなどのファイルを動的に生成可能
日本語の変数名が使用可能
- **OpenPyXL**
Excelファイルの読み書きが可能
`名前の定義` 機能を使用

## 例

元のExcelパラメータシート

![](/images/20241103_jinja-cloudformation/1.png)

Jinjaテンプレート
- if文
  - EBSのIOPSは、gp2の場合は指定できず、gp3の場合に指定可能です
- for文
  - Tagの数は可変であるため、動的に処理できるようにしています

![](/images/20241103_jinja-cloudformation/2.png)

生成されるCloudFormationテンプレート
![](/images/20241103_jinja-cloudformation/3.png)

## 作成手順

1. 普通にExcelパラメータシートを作成
2. 読み取りたいセルに `名前の定義` を設定
3. Jinjaテンプレートを作成
4. ExcelパラメータシートとJinjaテンプレートを読み取り、CloudFormationテンプレートを生成するPythonコードを作成

### 名前の定義

ExcelをPythonで読み取る際に、セル番号（例: 「A1」）を直接参照すると、Excelファイルの構造が少しでも変わると正しく値を取得できなくなります

これを避けるために、`名前の定義` を使用します

セルを範囲指定し、セル番号(例: 「A1」)が表示されているボックスに任意の名前を入力することで、セルの範囲に名前を設定できます

これにより、Pythonコード内でこの名前を使って安定したセル参照が可能になります

**メリット**

- セルの移動や行の追加・削除による影響を受けずに、常に同じ値を取得可能
- Tagなどの項目数が増減する場合でも、for文を用いることでPythonコードやJinjaテンプレートを変更せずに対応可能

設定例
![](/images/20241103_jinja-cloudformation/4.png)

`タグ` と `AMI` を移動しても影響ありません
![](/images/20241103_jinja-cloudformation/5.png)

空行を挿入して、セルの位置が変わっても影響ありません
![](/images/20241103_jinja-cloudformation/6.png)

読み取る値が増えても影響ありません
![](/images/20241103_jinja-cloudformation/7.png)

## コード

サンプルのExcelパラメータシートとPythonコードを格納しているので参考にしてください
Pythonコードは、100行くらいです

https://github.com/Flupinochan/excel-to-cloudformation

※補足
Excelファイルを読み込む際に、以下のように `data_only=True` オプションを付けること
これを設定しないと、`=` 等で値を参照している場合やExcel関数の式がそのまま取得されてしまいます

```python
excel_file_name = "【EC2】パラメータシート.xlsx"
wb = openpyxl.load_workbook(excel_file_name, data_only=True)
```