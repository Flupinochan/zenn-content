---
title: "AWS Pricing APIでCPU、Memory情報を取得"
emoji: "🙌"
type: "tech"
topics: ["RDS", "AWS"]
published: true
---

「アカウント内にある全ての RDS の CPU と Memory 情報を取得して欲しい」とお願いされました
list や describe 系の API で余裕で取得できると思いました
無理でした…(^▽^;)

https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/pricing.html

そのため、AWS サービスの詳細な価格情報を取得できる Pricing API を利用しました
価格だけでなく、細かいスペック(CPU や Memory 情報)も取得できます

## Pricing API で指定可能な AWS サービスコードを取得

```python
import boto3

def list_service_codes():
    # Pricing APIはus-east-1リージョンで利用可能
    pricing_client = boto3.client("pricing", region_name="us-east-1")
    service_codes = []
    paginator = pricing_client.get_paginator("describe_services")
    for page in paginator.paginate():
        services = page["Services"]
        for service in services:
            service_codes.append(service["ServiceCode"])
    print(service_codes)

if __name__ == "__main__":
    list_service_codes()
```

「AmazonEC2」や「AmazonRDS」が取得できました

## AmazonRDS の各インスタンスクラスの詳細情報を取得

```python
# 実行例
# python describe_service_price.py AmazonRDS

import boto3
import sys

def list_service_prices(service_code):
    pricing = boto3.client("pricing", region_name="us-east-1")
    paginator = pricing.get_paginator("get_products")
    # 東京リージョンの価格のみにフィルタリング
    for page in paginator.paginate(
        ServiceCode=service_code,
        Filters=[
            {
                "Type": "TERM_MATCH",
                "Field": "regionCode",
                "Value": "ap-northeast-1",
            },
        ],
    ):
        for price_item in page["PriceList"]:
            print(price_item)

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("引数にサービスコードを指定してください")
        sys.exit(1)
    service_code = sys.argv[1]
    list_service_prices(service_code)
```

出力内容抜粋

```json
{
  "product": {
    "productFamily": "Database Instance",
    "attributes": {
      "engineCode": "5",
      "instanceTypeFamily": "M5d",
      "memory": "128 GiB",
      "vcpu": "32",
      "instanceType": "db.m5d.8xlarge",
      "usagetype": "APN1-Multi-AZUsage:db.m5d.8xl",
      "locationType": "AWS Region",
      "storage": "2 x 600 NVMe SSD",
      "normalizationSizeFactor": "128",
      "instanceFamily": "General purpose",
      "databaseEngine": "Oracle",
      "databaseEdition": "Enterprise",
      "regionCode": "ap-northeast-1",
      "servicecode": "AmazonRDS",
      "physicalProcessor": "Intel Xeon Platinum 8175",
      "licenseModel": "Bring your own license",
      "currentGeneration": "Yes",
      "networkPerformance": "10 Gbps",
      "deploymentOption": "Multi-AZ",
      "location": "Asia Pacific (Tokyo)",
      "servicename": "Amazon Relational Database Service",
      "processorArchitecture": "64-bit",
      "operation": "CreateDBInstance:0005"
    },
    "sku": "22GTWH3M7MMRFZQ9"
  },
  "serviceCode": "AmazonRDS",
  "terms": {
    "OnDemand": {
      "22GTWH3M7MMRFZQ9.JRTCKXETXF": {
        "priceDimensions": {
          "22GTWH3M7MMRFZQ9.JRTCKXETXF.6YS6EN2CT7": {
            "unit": "Hrs",
            "endRange": "Inf",
            "description": "$ 8.851 per RDS db.m5d.8xlarge Multi-AZ instance hour (or partial hour) running Oracle EE (BYOL)",
            "appliesTo": [],
            "rateCode": "22GTWH3M7MMRFZQ9.JRTCKXETXF.6YS6EN2CT7",
            "beginRange": "0",
            "pricePerUnit": {
              "USD": "8.8510000000"
            }
          }
        },
        "sku": "22GTWH3M7MMRFZQ9",
        "effectiveDate": "2024-08-01T00:00:00Z",
        "offerTermCode": "JRTCKXETXF",
        "termAttributes": {}
      }
    },
    "Reserved": {
      // Reserved pricing options are included here
    }
  },
  "version": "20240830162724",
  "publicationDate": "2024-08-30T16:27:24Z"
}
```

## アカウント内の各 RDS の CPU、Memory 情報を取得

```python
import sys
import os
import logging
import json
from time import struct_time
from datetime import datetime
from zoneinfo import ZoneInfo

import pandas as pd
import boto3
from botocore.config import Config
from openpyxl import Workbook
from openpyxl.utils.dataframe import dataframe_to_rows

##################
# ディレクトリ作成 #
##################
try:
    output_directory = "list_rds"  # 出力ディレクトリ
    if not os.path.exists(output_directory):
        os.makedirs(output_directory)
except Exception as e:
    raise Exception(f"ディレクトリ作成エラー: {e}")

####################
# グローバル定数定義 #
####################
try:
    # boto3 client設定
    config = Config(
        retries={"max_attempts": 30, "mode": "standard"},
        read_timeout=900,
        connect_timeout=900,
        region_name="ap-northeast-1",
    )
    # 可変パラメータ
    jst = ZoneInfo("Asia/Tokyo")
    current_date = datetime.now(jst).strftime("%Y-%m-%d")
    output_excel = os.path.join(output_directory, "rds_details.xlsx")  # 出力Excelファイル名
    output_csv = os.path.join(output_directory, "rds_details.csv")  # 出力CSVファイル名
    log_file = os.path.join(output_directory, f"{current_date}.log")  # 出力ログファイル名
except Exception as e:
    raise Exception(f"グローバル定数定義エラー: {e}")

#############
# ロガー設定 #
#############
try:

    def custom_time(*args) -> struct_time:
        return datetime.now(jst).timetuple()

    log_level = logging.INFO  # ログレベル
    log_format = "[%(asctime)s.%(msecs)03d JST] [%(levelname)s] [%(lineno)d行目] [関数名: %(funcName)s] %(message)s"
    date_format = "%Y-%m-%d %H:%M:%S"
    formatter = logging.Formatter(log_format, datefmt=date_format)
    formatter.converter = custom_time

    stdout_handler = logging.StreamHandler(sys.stdout)
    stdout_handler.setLevel(log_level)
    file_handler = logging.FileHandler(log_file, mode="a")
    file_handler.setLevel(log_level)

    stdout_handler.setFormatter(formatter)
    file_handler.setFormatter(formatter)

    logger = logging.getLogger("custom_logger")
    logger.addHandler(stdout_handler)
    logger.addHandler(file_handler)
    logger.setLevel(log_level)
except Exception as e:
    raise Exception(f"ロガー設定エラー: {e}")


###########
# 処理開始 #
###########
# Pricing APIから対象のRDSインスタンスクラスのCPUとメモリ情報を取得する関数
def get_instance_specs(instance_class):
    specs = {"vCPUs": "Unknown", "MemoryGiB": "Unknown"}
    try:
        pricing = boto3.client("pricing", region_name="us-east-1")  # Pricing APIはus-east-1リージョンで利用
        response = pricing.get_products(
            ServiceCode="AmazonRDS",
            Filters=[
                {"Type": "TERM_MATCH", "Field": "instanceType", "Value": instance_class},
                {"Type": "TERM_MATCH", "Field": "location", "Value": "Asia Pacific (Tokyo)"},
            ],
            MaxResults=1,
        )
        if response["PriceList"]:
            price_item = json.loads(response["PriceList"][0])
            attributes = price_item["product"]["attributes"]
            # vCPUs情報を取得
            if "vcpu" in attributes:
                specs["vCPUs"] = int(attributes["vcpu"])
            # Memory情報を取得
            if "memory" in attributes:
                memory_str = attributes["memory"]
                memory_value = float(memory_str.split()[0])
                specs["MemoryGiB"] = memory_value
        return specs
    except Exception as e:
        logger.error(f"RDSインスタンスクラス情報の取得でエラーが発生しました: {e}")
        return specs


# メイン処理
def list_rds_instances():
    rds_list = []
    try:
        # RDSインスタンスをlistする
        rds_client = boto3.client("rds", config=config)
        paginator = rds_client.get_paginator("describe_db_instances")
        for page in paginator.paginate():
            for db_instance in page["DBInstances"]:
                # RDSの情報を取得
                db_instance_identifier = db_instance.get("DBInstanceIdentifier", "")
                db_instance_class = db_instance.get("DBInstanceClass", "")
                engine = db_instance.get("Engine", "")
                # RDSのCPU、Memoryを取得
                specs = get_instance_specs(db_instance_class)
                # リストに追加
                rds_list.append(
                    {
                        "DBInstanceIdentifier": db_instance_identifier,
                        "Engine": engine,
                        "DBInstanceClass": db_instance_class,
                        "vCPUs": specs["vCPUs"],
                        "MemoryGiB": specs["MemoryGiB"],
                    }
                )
                logger.info(f'RDS情報取得 インスタンス名: {db_instance_identifier} CPU: {specs["vCPUs"]} Memory: {specs["MemoryGiB"]}')
    except Exception as e:
        logger.error(f"RDS情報取得処理にエラーが発生しました: {e}")
    # Pandas DataFrameに変換
    df = pd.DataFrame(rds_list)
    df = df.astype(str)
    return df


# 出力処理
def export_to_excel_and_csv(df):
    try:
        # Excel出力
        wb = Workbook()
        ws = wb.active
        ws.title = "RDS Details"
        for r in dataframe_to_rows(df, index=False, header=True):
            ws.append(r)
        wb.save(output_excel)
        logger.info(f"Excel出力ファイル名: {output_excel}")
        # CSV出力
        df.to_csv(output_csv, index=False)
        logger.info(f"CSV出力ファイル名: {output_csv}")
    except Exception as e:
        logger.error(f"Excel/CSVファイル出力でエラーが発生しました: {e}")


def main():
    df = list_rds_instances()
    export_to_excel_and_csv(df)


if __name__ == "__main__":
    main()
```

出力 Excel ファイル例
![](/images/20240901_aws-rds-spec/1.png)

Pricing API なめてました…
