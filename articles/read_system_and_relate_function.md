---
title: "参照系の処理における工夫とRelate関数"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["設計"]
published: false
---

- repositoryを中心として
- idをinterfaceで受ける。エラーを想定して
- returnはsliceにする。単一の値は、受ける側で絞る。いちいち面倒なのと、N+1を起こさないため
- relateで紐づけるが、集約内はmodelでrepositoryで、集約間はoutputでcontrollerで
- storeというsql詳細を知る実装は別なので呼び出す
- あくまでmodelを扱う。storeは本来はrecordというdbテーブルを返すが、modelで十分なときはmodel。基本はrecord不要。

