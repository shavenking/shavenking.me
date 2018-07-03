---
title: "Laravel 操作記錄資料庫設計、案例分析"
date: 2018-05-27T23:27:22+08:00
---

最近碰到一個案子，需求是記錄系統各 Model 的操作記錄，比如說：

1. 使用者更改暱稱
2. 錢包交易記錄（購買遊戲寵物、寵物，或是玩家間交易）
3. 道具使用記錄

如果撇除第一點，第二、三點之間其實是有連貫的關係，也就是使用者在看操作記錄時，應預期會長這樣：

- `2018-01-01 00:00:00` 我的錢包少了 1000 元
- `2018-01-01 00:00:00` 我買了遊戲寵物，並贈送給使用者 B

此時使用者 B 的操作記錄會有

- `2018-01-01 00:00:00` 收到禮物：遊戲寵物（來自使用者 A）

總結來說，整個操作記錄必須是連貫的，並且需要記錄整個系統中的 Model，而不是只有錢包的餘額，以這樣的需求出發，首先想到的設計，就是把所有的 Model 操作記錄都存在同一張資料表，如此一來透過條件篩選時，就能輕易組織出某區間內特定 Model 的操作記錄。

## 問題一：如何使用同一種資料表，記錄各種不同 Model 的變化記錄？

[Polymorphic Relations](https://laravel.com/docs/5.6/eloquent-relationships#polymorphic-relations)，Laravel 提供了一個很好的設計，讓我們能輕鬆地使用同一個 Model 對應多個 Model Relation。

以上述需求為例，我們可以建立一個 Model 名為：OperationHistory。其 Migration 檔案如下：

    Schema::create('operation_histories', function (Blueprint $table) {
        $table->increments('id');
        $table->morphs('operatable');
        $table->unsignedInteger('operator_id')->index()->nullable();
        $table->unsignedInteger('user_id')->index()->nullable();
        $table->unsignedTinyInteger('type')->index();
        $table->json('result_data');
        $table->timestamps();
    });

- 其中 `$table→morphs` 即為 Laravel 提供的 Polymorphic Relations 對應的欄位，其原理就是透過建立 type、id 來儲存這筆記錄對應的 Model。

- `operator_id`、`user_id`：即為 Model 兩邊使用者的對應關係，比如說，A 購買了遊戲寵物並贈送給 B，此時 `operator_id` 為 A（A 啟動了這個動作），`user_id` 為 B（B 為這個動作的附加使用者）。
- `type`：則是記錄這個動作的「種類」，可能為「新增」、「修改」、「激活」、「交易」等等，視業務需求而定。
- `result_data`：代表的是動作「結束後」，Model 最終的資料，因為 Laravel Model 能夠直接 JSON 化，所以此欄位設計成 JSON 欄位，如果需要可以被搜尋，則需要做進一步調整，但不在此篇範圍內。

有了這樣的設計，就能在同一張表中，記錄各種 Model 的操作記錄，即使將來有新的 Model 出現，也能夠在不更動資料庫結構的狀況下，記錄在資料庫裡面。

## 問題二：如果業務需求希望界面能夠顯示變化量怎麼辦？

什麼是變化量？舉例來說，錢包可能會需要顯示「該次操作」，錢包轉出了多少錢。

### 解法一：業務需求真的需要顯示變化量嗎？

是否考慮過，使用者在檢視「所有」操作記錄時，真的在意每筆的變化量嗎？如果使用者能夠搜尋操作記錄，難道使用者不能自己看前後兩筆紀錄，來計算出變化量嗎？

### 解法二：分頁

我們可以給使用者篩選操作記錄的功能，比如顯示所有「錢包」的操作記錄，此時一定會有分頁，所以在有分頁的狀況下，直接計算每筆操作記錄的變化量，對伺服器的負擔是可以接受的。

## 問題三：如果需要顯示大量的操作記錄，是否還能在程式中，動態計算每筆操作記錄對應的變化量？

可以！

首先如果是需要顯示上千、數萬筆操作記錄的餘額，還是可以透過分頁的方式解決，想象一下折線圖表，看起來是顯示所有記錄，但其實還是透過分頁，分批畫上去，在使用者左滑、右滑時，才動態載入。

但是不可能永遠都能用分頁的方式解決，如果需求還是需要快速取得所有變化量，那就不能使用這樣的設計，沒有一種架構設計能夠應付各種業務需求。

## 問題四：如果這個方法這麼多缺點，那為什麼不要一開始就設計能夠記錄變化量的資料結構？

Laravel 的 ORM 設計是 Active Record，任何資料的更新，在過程中都必須另外保留下來，相反地，最終的資料結果反而是容易保留的，所以上述的架構設計，能夠讓「記錄操作記錄」這件事情，變得很單純，不需要小心翼翼的保留過程中的變化量，所以在謹慎考慮過業務需求後，還是決定使用這樣的資料結構。

## 問題五：如果用 JSON 欄位記錄當時的 Model 資料，未來有新增欄位時是否需補上？

假設有一個 Model 叫做 Dragon，其資料結構如下：

```
id
owner_id
created_at
updated_at
```

問題：可想而知，OperationHistory 的 `result_data` 會記錄以上四個欄位，問題來了，如果今天需要為這個 Model 加上 `type` 欄位，用以標示 Dragon 的種類，就會造成新舊資料不同步的問題，舊有的 OperationHistory 中只有四個屬性，新建資料多了 `type` 欄位，造成同一種 OperationHistory 會有不同的格式（Key 不同視為不同格式）。

### 解法一：補資料

1. 做法：寫程式讀資料庫的資料，為所有需要補上的資料做處理。
2. 優點：修改欄位於無形之間（對 API 使用者友好）。
3. 缺點：想象一下如果一個經營許久的網站，OperationHistory 的數量驚人，為了補資料，可能需要寫一個背景程式，一點一滴的把資料補回來。

### 解法二：補資料（動態）

1. 做法：在 API 回傳之前，在程式中檢查，如果是 Dragon 的操作記錄，就補上 `type` 欄位。
2. 優點：不需要設計「解法一」中補資料的程式、流程。
3. 缺點：程式每次都要掃過要吐出去的資料。
4. 備註：因為 API 有分頁的狀況下，所以掃過的資料量不大。

### 解法三：通知所有 API 使用者，需對沒有欄位的資料做處理

1. 做法：如題。
2. 優點：後端躺著爽爽賺。
3. 缺點：資料處理邏輯（沒有 `type` 就代表是 XXX 的邏輯），被轉移到前端，可能造成各前端不同步（Web、APP 等等），未來有調整會直接波及到所有 API 使用者。
