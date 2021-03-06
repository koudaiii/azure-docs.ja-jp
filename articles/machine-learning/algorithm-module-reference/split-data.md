---
title: 'Split Data (データの分割): モジュール リファレンス'
titleSuffix: Azure Machine Learning
description: Azure Machine Learning で Split Data (データの分割) モジュールを使用して、データセットを 2 つの異なるセットに分割する方法について説明します。
services: machine-learning
ms.service: machine-learning
ms.subservice: core
ms.topic: reference
author: likebupt
ms.author: keli19
ms.date: 10/22/2019
ms.openlocfilehash: a4c93b12ad654e54a7f3c7ee0e75507d5cb45e90
ms.sourcegitcommit: 829d951d5c90442a38012daaf77e86046018e5b9
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 10/09/2020
ms.locfileid: "90907818"
---
# <a name="split-data-module"></a>Split Data (データの分割) モジュール

この記事では Azure Machine Learning デザイナーのモジュールについて説明します。

Split Data モジュールを使用して、データセットを 2 つの異なるセットに分割します。

このモジュールは、データをトレーニングとテストのセットに分割する必要がある場合に便利です。 また、データの分割方法をカスタマイズすることもできます。 一部のオプションでは、データのランダム化がサポートされます。 その他のものは、特定のデータ型またはモデルの種類に合わせて調整されます。

## <a name="configure-the-module"></a>モジュールを構成する

> [!TIP]
> 分割モードを選択する前に、すべてのオプションに目を通して、必要な分割の種類を決定してください。
> 分割モードを変更すると、他のすべてのオプションがリセットされる可能性があります。

1. デザイナーで、**Split Data (データの分割)** モジュールをパイプラインに追加します。 このモジュールは、 **[Data Transformation]\(データ変換\)** の **[Sample and Split]\(サンプルおよび分割\)** カテゴリにあります。

1. **Splitting mode (分割モード)** : 用意しているデータの種類と分割方法に応じて、次のモードのいずれかを選択します。 各分割モードには、さまざまなオプションがあります。

   - **Split Rows (行の分割)** : データを 2 つの部分に分割する場合は、このオプションを使用します。 各分割に含めるデータの割合を指定できます。 既定では、データは 50 / 50 に分割されます。

     また、各グループでの行の選択をランダム化し、層化サンプリングを使用することもできます。 層化サンプリングでは、2 つの結果データセット間で値を均等に分配する対象とする 1 つのデータ列を選択する必要があります。  

   - **Regular Expression Split (正規表現分割)** : 1 つの列の値をテストしてデータセットを分割する場合は、このオプションを選択します。

     たとえば、センチメントを分析している場合は、テキスト フィールドに特定の製品名があるかどうかを確認できます。 その後、対象製品名を含む行と対象製品名を含まない行にデータセットを分割することができます。

   - **Relative Expression Split (相対表現分割)** : 数値の列に条件を適用する場合は常に、このオプションを使用します。 この数値には、日付/時刻フィールド、年齢や金額を含む列、またはパーセンテージを指定できます。 たとえば、アイテムのコストに基づいてデータセットを分割したり、年齢範囲別にユーザーをグループ化したり、カレンダー日付別にデータを分類したりすることができます。

### <a name="split-rows"></a>行の分割

1. デザイナーで [Split Data](./split-data.md) モジュールをパイプラインに追加し、分割するデータセットを接続します。
  
1. **[Splitting mode]\(分割モード\)** として、 **[Split rows]\(行の分割\)** を選択します。 

1. **Fraction of rows in the first output dataset (最初の出力データセット内の行の割合)** : このオプションを使用して、最初 (左側) の出力に送る行数を決定します。 その他の行はすべて、2 番目 (右側) の出力に送られます。

   比率は、最初の出力データセットに送信される行の割合を示します。したがって、0 から 1 までの 10 進数を入力する必要があります。
     
   たとえば、値として **0.75** を入力した場合、データセットは 75/25 に分割されます。 この分割では、行の 75% が最初の出力データセットに送信されます。 残りの 25% は 2 番目の出力データセットに送信されます。
  
1. 2 つのグループに送信するデータの選択をランダム化する場合は、 **[Randomized spli]\(ランダム分割\)** オプションを選択します。 これは、トレーニングとテストのデータセットを作成するときに推奨されるオプションです。

1. **Random seed (ランダム シード)** : 負でない整数値を入力して、使用するインスタンスの擬似乱数シーケンスを開始します。 この既定のシードは、乱数を生成するすべてのモジュールで使用されます。 

   シードを指定すると、結果は再現可能になります。 分割操作の結果を繰り返す必要がある場合は、乱数ジェネレーターに対してシードを指定する必要があります。 それ以外の場合、ランダム シードは既定で **0** に設定されます。これは、初期シード値がシステム クロックから取得されることを意味します。 その結果、分割を実行するたびに、データの配分が若干異なる可能性があります。 

1. **Stratified split (階層分割)** :このオプションを **[True]** に設定すると、2 つの出力データセットには、"*階層列*" または "*階層キー列*" 内の代表的なサンプル値が確実に含められます。 

   階層サンプリングの場合、各出力データセットに含められる各ターゲット値の割合がほぼ同じになるようにデータが分割されます。 たとえば、トレーニングとテストのセットを、結果やその他の列 (性別など) に関してほぼバランスが取れた状態にしたい場合があります。

1. パイプラインを送信します。


## <a name="select-a-regular-expression"></a>正規表現を選択する

1. パイプラインに [Split Data](./split-data.md) モジュールを追加し、それを、分割するデータセットの入力として接続します。  
  
1. **[Splitting mode]\(分割モード\)** として、 **[Regular expression split]\(正規表現分割\)** を選択します。

1. **[Regular expression]\(正規表現\)** ボックスに、有効な正規表現を入力します。 
  
   正規表現は、Python の正規表現の構文に従う必要があります。

1. パイプラインを送信します。

   指定した正規表現に基づき、データセットは 2 つの行セット (式に一致する値を含む行と残りのすべての行) に分割されます。 

次の例は、 **[正規表現]** オプションを使用してデータセットを分割する方法を示しています。 

### <a name="single-whole-word"></a>1 単語単位 

この例では、列 `Text` にテキスト `Gryphon` を含むすべての行は最初のデータセットに入れられます。 他の行は、**Split Data** の 2 番目の出力に入れられます。

```text
    \"Text" Gryphon  
```

### <a name="substring"></a>Substring

この例では、データセットの 2 番目の列内の任意の位置で、指定された文字列を検索します。 ここでは、位置はインデックス値 1 で示されています。 照合では大文字と小文字が区別されます。

```text
(\1) ^[a-f]
```

最初の結果データセットには、インデックス列が `a`、`b`、`c`、`d`、`e`、`f` のいずれかの文字から始まるすべての行が含まれます。 他のすべての行は、2 番目の出力に送られます。

## <a name="select-a-relative-expression"></a>相対表現を選択する

1. パイプラインに [Split Data](./split-data.md) モジュールを追加し、それを、分割するデータセットの入力として接続します。
  
1. **[Splitting mode]\(分割モード\)** として、 **[Relative Expression]\(相対表現\)** を選択します。
  
1. **[Relational Expression]\(関係式\)** ボックスに、単一の列に対して比較演算を実行する式を入力します。

   **数値の列**の場合:
   - 列には、日付および時刻データ型など、任意の数値データ型の数値が含まれています。
   - 式では、1 つの列名の最大値を参照できます。
   - AND 演算にはアンパサンド文字 `&` を使用します。 OR 演算にはパイプ文字 `|` を使用します。
   - サポートされている演算子は、`<`、`>`、`<=`、`>=`、`==`、`!=` です。
   - `(` と `)` を使用して、演算をグループ化することはできません。
   
   **文字列の列の場合**:
   - `==`、`!=` の演算子がサポートされています。

1. パイプラインを送信します。

   式によってデータセットは 2 つの行セットに分割されます。一方は、条件と一致する値を含む行のセット、もう一方は残りのすべての行のセットです。

次の例は、**Split Data** モジュールの **[相対表現]** オプションを使用してデータセットを分割する方法を示しています。  

### <a name="calendar-year"></a>暦年

一般的なシナリオは、データセットを年で分割することです。 次の式を使うと、列 `Year` の値が `2010` より大きいすべての行が選択されます。

```text
\"Year" > 2010
```

日付式では、データ列に含まれているすべての日付部分を考慮する必要があります。 データ列の日付の形式は一貫している必要があります。 

たとえば、形式 `mmddyyyy` を使用する日付列では、式は次のようになります。

```text
\"Date" > 1/1/2010
```

### <a name="column-index"></a>列インデックス

次の式は、列インデックスを使用して、データセットの最初の列から 30 以下の値 (20 を除く) を含むすべての列を選択する方法を示しています。

```text
(\0)<=30 & !=20
```


## <a name="next-steps"></a>次のステップ

Azure Machine Learning で[使用できる一連のモジュール](module-reference.md)を参照してください。 
