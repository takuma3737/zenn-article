---
title: "DB設計の基礎~具体例で学ぶ正規化~"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["DB設計", "正規化", "初学者"]
published: true
---

# はじめに
はじめまして！いずみんです！新卒半年目になるのですが、恥ずかしながらデータベースの論理設計の基本である、**正規化**について体系的に学んだことがありませんでした😇
そこで実際にドメインを設定~テーブルの正規化まで学んだので,「正規化って何それおいしいの？」「第一正規化と第二正規化の違いがよくわからない...」という方向けに、**具体的なテーブル例**を使って正規化の手順を解説します！

この記事では、オンライン学習プラットフォームという身近なドメインを例に、非正規化されたテーブルから第三正規化までの過程をステップバイステップで見ていきます。

## 📚 正規化とは？

正規化とは、**データベースのテーブル設計において、データの冗長性を排除し、整合性を保ちやすくする手法**です。

正規化には段階があり、主に以下の3つの正規形があります
- **第一正規化**：繰り返し項目を別レコードとして独立
- **第二正規化**：部分関数従属を排除
- **第三正規化**：推移的関数従属を排除

それでは、実際の例を見ながら理解していきましょう！

# 🎯 ドメインの設定

**オンライン学習プラットフォームの受講管理システム**

- 受講者（Student）は複数のコース（Course）を受講できる
- 各コースには講師（Instructor）が1名担当している
- 各コースには複数のコンテンツ（Content）がある（動画、テスト等）
- 受講者はコースごとに受講開始日・完了日を持つ
- 受講者はコンテンツごとに受講履歴（完了日時）を持つ

:::message
複雑にしたくないので簡略化した定義をしています。ご容赦ください。
:::

上記の定義のもと、以下のような管理表が考えられます。

**【正規化前のテーブル】** ※1つのセルに複数の値が入っている状態

| student_id | student_name | student_email | course_title | instructor_name | instructor_email | started_at | finished_at | content_title | content_type | started_at | finished_at |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| s1 | 田中太郎 | tanaka@example.com | Go入門<br>SQL入門 | 山田先生<br>佐藤先生 | yamada@example.com<br>sato@example.com | 2025/10/15<br>2025/10/16 | 2025/10/18<br>null | Goの基礎<br>Goの応用<br>SQLの基礎<br>SQLの応用 | video<br>test<br>video<br>test | 2025/10/15<br>2025/10/15<br>2025/10/16<br>2025/10/16 | 2025/10/15<br>2025/10/17<br>2025/10/16<br>null |
| s2 | 鈴木花子 | suzuki@example.com | SQL入門 | 佐藤先生 | sato@example.com | 2025/10/17 | 2025/10/19 | SQLの基礎<br>SQLの応用 | video<br>test | 2025/10/17<br>2025/10/17 | 2025/10/18<br>2025/10/19 |

このように**1つのセルに複数の値が入っている状態**は、以下の問題があります
- データの一貫性を保つのが困難
- クエリが複雑になる

上記テーブルの正規化を段階的に行うことで、これらの問題を解決していきます。

## 第一正規化：繰り返し項目を別レコードとして独立

**目的**：各カラムに単一の値のみを格納します。

上記テーブルを第一正規化した結果は以下のようになります。

| student_id | student_name | student_email | course_title | instructor_name | instructor_email | started_at | finished_at | content_title | content_type | started_at | finished_at |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| s1 | 田中太郎 | tanaka@example.com | Go入門 | 山田先生 | yamada@example.com | 2025/10/15 | 2025/10/18 | Goの基礎 | video | 2025/10/15 | 2025/10/15 |
| s1 | 田中太郎 | tanaka@example.com | Go入門 | 山田先生 | yamada@example.com | 2025/10/15 | 2025/10/18 | Goの応用 | test | 2025/10/15 | 2025/10/17 |
| s1 | 田中太郎 | tanaka@example.com | SQL入門 | 佐藤先生 | sato@example.com | 2025/10/16 | null | SQLの基礎 | video | 2025/10/16 | 2025/10/16 |
| s1 | 田中太郎 | tanaka@example.com | SQL入門 | 佐藤先生 | sato@example.com | 2025/10/16 | null | SQLの応用 | test | 2025/10/16 | null |
| s2 | 鈴木花子 | suzuki@example.com | SQL入門 | 佐藤先生 | sato@example.com | 2025/10/17 | 2025/10/19 | SQLの基礎 | video | 2025/10/17 | 2025/10/18 |
| s2 | 鈴木花子 | suzuki@example.com | SQL入門 | 佐藤先生 | sato@example.com | 2025/10/17 | 2025/10/19 | SQLの応用 | test | 2025/10/17 | 2025/10/19 |

1つのセルに複数の値が入っていた部分を分割し、各行に1つずつ格納しました。
しかし、**データの重複が大量に発生**しています（student_nameやinstructor_emailなど）。

##  第二正規化：部分関数従属性の排除

**目的**：主キーの一部にのみ従属する列（部分関数従属）を排除し、各テーブルに適切な主キーを割り当てます。

:::message
部分関数従属ってなんやねん！と思った方は[こちらの動画](https://www.youtube.com/watch?v=zcLCZKOAOjE)がとても分かりやすいです。
:::


### ステップ1：student、course、contentにざっくり分割

まず第一正規化されたテーブルを見て、繰り返されている情報ごとにテーブルを分割してみます。

**【Studentsテーブル】**

| student_id (PK) | student_name | student_email | course_id (FK) | started_at | finished_at |
| --- | --- | --- | --- | --- | --- |
| s1 | 田中太郎 | tanaka@example.com | c1 | 2025/10/15 | 2025/10/18 |
| s1 | 田中太郎 | tanaka@example.com | c2 | 2025/10/16 | null |
| s2 | 鈴木花子 | suzuki@example.com | c2 | 2025/10/17 | 2025/10/19 |

**【Coursesテーブル】**

| course_id (PK) | course_title | instructor_name | instructor_email |
| --- | --- | --- | --- |
| c1 | Go入門 | 山田先生 | yamada@example.com |
| c2 | SQL入門 | 佐藤先生 | sato@example.com |

**【Contentsテーブル】**

| content_id (PK) | course_id (FK) | content_title | content_type | student_id (FK) | started_at | finished_at |
| --- | --- | --- | --- | --- | --- | --- |
| ct1 | c1 | Goの基礎 | video | s1 | 2025/10/15 | 2025/10/15 |
| ct2 | c1 | Goの応用 | test | s1 | 2025/10/15 | 2025/10/17 |
| ct3 | c2 | SQLの基礎 | video | s1 | 2025/10/16 | 2025/10/16 |
| ct4 | c2 | SQLの応用 | test | s1 | 2025/10/16 | null |
| ct5 | c2 | SQLの基礎 | video | s2 | 2025/10/17 | 2025/10/18 |
| ct6 | c2 | SQLの応用 | test | s2 | 2025/10/17 | 2025/10/19 |


上記のテーブルには以下の問題があります

**Studentsテーブルに`course_id`、`started_at`、`finished_at`が含まれている**
- これらは「受講者の属性」ではなく「受講履歴」の情報
- 同じ受講者が複数のコースを受講すると、受講者情報が重複する

**Contentsテーブルに`student_id`、`started_at`、`finished_at`が含まれている**
- これらは「コンテンツの属性」ではなく「学習履歴」の情報
- コンテンツの定義と学習進捗が混在している

### 2. マスターデータと履歴データを分離

**マスターデータ**（基本情報）と**履歴データ**を分けます。

**Studentsテーブル**：受講者の基本情報のみ

| student_id (PK) | student_name | student_email |
| --- | --- | --- |
| s1 | 田中太郎 | tanaka@example.com |
| s2 | 鈴木花子 | suzuki@example.com |

**Coursesテーブル**：コースの基本情報のみ

| course_id (PK) | course_title | instructor_name | instructor_email |
| --- | --- | --- | --- |
| c1 | Go入門 | 山田先生 | yamada@example.com |
| c2 | SQL入門 | 佐藤先生 | sato@example.com |

**Contentsテーブル**：コンテンツの基本情報のみ

| content_id (PK) | course_id (FK) | content_title | content_type |
| --- | --- | --- | --- |
| ct1 | c1 | Goの基礎 | video |
| ct2 | c1 | Goの応用 | test |
| ct3 | c2 | SQLの基礎 | video |
| ct4 | c2 | SQLの応用 | test |

**Course_Historiesテーブル**：コース受講履歴（新規追加）

| course_history_id (PK) | student_id (FK) | course_id (FK) | started_at | finished_at |
| --- | --- | --- | --- | --- |
| ch1 | s1 | c1 | 2025/10/15 | 2025/10/18 |
| ch2 | s1 | c2 | 2025/10/16 | null |
| ch3 | s2 | c2 | 2025/10/17 | 2025/10/19 |

**Content_Historiesテーブル**：コンテンツ学習履歴（新規追加）

| content_history_id (PK) | student_id (FK) | content_id (FK) | started_at | finished_at |
| --- | --- | --- | --- | --- |
| cth1 | s1 | ct1 | 2025/10/15 | 2025/10/15 |
| cth2 | s1 | ct2 | 2025/10/15 | 2025/10/17 |
| cth3 | s1 | ct3 | 2025/10/16 | 2025/10/16 |
| cth4 | s1 | ct4 | 2025/10/16 | null |
| cth5 | s2 | ct3 | 2025/10/17 | 2025/10/18 |
| cth6 | s2 | ct4 | 2025/10/17 | 2025/10/19 |

これにより部分関数従属が解消され、データの重複が大幅に削減されました。
しかし、まだ**推移的関数従属**の問題が残っています。



## 第三正規化：推移的関数従属性の排除

**目的:** 主キー以外の列に従属する列(推移的関数従属)を排除します。

先ほどのCoursesテーブルを見てみましょう

| course_id | course_title | instructor_name | instructor_email |
| --- | --- | --- | --- |
| c1 | Go入門 | 山田先生 | yamada@example.com |
| c2 | SQL入門 | 佐藤先生 | sato@example.com |

この場合、`instructor_email` は主キー `course_id` に従属していますが、実際には `instructor_name` に従属しています。
つまり、`course_id → instructor_name → instructor_email` という**推移的関数従属**が存在します。

:::message
**問題点**
❌ 山田先生が複数のコースを担当する場合、同じメールアドレスが繰り返される
❌ 山田先生のメールアドレスが変更された場合、複数のレコードを更新する必要がある
:::

第三正規化により、講師情報を別テーブルに分割します

**【Studentsテーブル】**（変更なし、主キー：student_id）

| student_id (PK) | student_name | student_email |
| --- | --- | --- |
| s1 | 田中太郎 | tanaka@example.com |
| s2 | 鈴木花子 | suzuki@example.com |

**【Instructorsテーブル】**（新規追加、主キー：instructor_id）

| instructor_id (PK) | instructor_name | instructor_email |
| --- | --- | --- |
| i1 | 山田先生 | yamada@example.com |
| i2 | 佐藤先生 | sato@example.com |

**【Coursesテーブル】**（instructor情報を外部キーに変更、主キー：course_id）

| course_id (PK) | course_title | instructor_id (FK) |
| --- | --- | --- |
| c1 | Go入門 | i1 |
| c2 | SQL入門 | i2 |

**【Contentsテーブル】**（変更なし、主キー：content_id）

| content_id (PK) | course_id (FK) | content_title | content_type |
| --- | --- | --- | --- |
| ct1 | c1 | Goの基礎 | video |
| ct2 | c1 | Goの応用 | test |
| ct3 | c2 | SQLの基礎 | video |
| ct4 | c2 | SQLの応用 | test |

**【Course_Historiesテーブル】**（変更なし、主キー：course_history_id）

| course_history_id (PK) | student_id (FK) | course_id (FK) | started_at | finished_at |
| --- | --- | --- | --- | --- |
| ch1 | s1 | c1 | 2025/10/15 | 2025/10/18 |
| ch2 | s1 | c2 | 2025/10/16 | null |
| ch3 | s2 | c2 | 2025/10/17 | 2025/10/19 |

**【Content_Historiesテーブル】**（変更なし、主キー：content_history_id）

| content_history_id (PK) | student_id (FK) | content_id (FK) | started_at | finished_at |
| --- | --- | --- | --- | --- |
| cth1 | s1 | ct1 | 2025/10/15 | 2025/10/15 |
| cth2 | s1 | ct2 | 2025/10/15 | 2025/10/17 |
| cth3 | s1 | ct3 | 2025/10/16 | 2025/10/16 |
| cth4 | s1 | ct4 | 2025/10/16 | null |
| cth5 | s2 | ct3 | 2025/10/17 | 2025/10/18 |
| cth6 | s2 | ct4 | 2025/10/17 | 2025/10/19 |

これで推移的関数従属が解消され、第三正規化が完了しました。

# 📝まとめ

正規化のプロセス

1. **第一正規化**：繰り返し項目を排除し、各セルに単一の値のみを格納
2. **第二正規化**：部分関数従属を排除し、主キー全体に従属する列のみを残す
3. **第三正規化**：推移的関数従属を排除し、主キー以外の列への従属を解消


実際のシステム設計では、**正規化の原則を理解した上で、ビジネス要件やパフォーマンス要件に応じて適切なバランスを取る**ことが重要です。
