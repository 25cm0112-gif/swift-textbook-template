# 第8章：ウィジェット

> 執筆者：呉迎豪
> 最終更新：2026-7-8

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

例：この章では、WidgetKitを使ってホーム画面やロック画面に表示できるウィジェットを実装する方法を学ぶ。具体的には毎日異なる名言を表示するウィジェットを題材にして、TimelineProviderの仕組み、ウィジェットビューの構成、複数サイズへの対応、そしてメインアプリとの連携方法を学ぶ。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
mport SwiftUI

// MARK: - 名言データ（アプリとウィジェットで共有）

struct Quote: Identifiable, Codable {
    let id: Int
    let text: String
    let author: String
}

struct QuoteStore {
    static let quotes: [Quote] = [
        Quote(id: 1, text: "為せば成る、為さねば成らぬ何事も", author: "上杉鷹山"),
        Quote(id: 2, text: "千里の道も一歩から", author: "老子"),
        Quote(id: 3, text: "継続は力なり", author: "ことわざ"),
        Quote(id: 4, text: "失敗は成功のもと", author: "ことわざ"),
        Quote(id: 5, text: "知ることは愛することの始まりである", author: "ことわざ"),
        Quote(id: 6, text: "学びて思わざれば則ち罔し", author: "孔子"),
        Quote(id: 7, text: "過ちて改めざる、是を過ちと謂う", author: "孔子"),
    ]

    static func todaysQuote() -> Quote {
        let dayOfYear = Calendar.current.ordinality(of: .day, in: .year, for: Date()) ?? 0
        let index = dayOfYear % quotes.count
        return quotes[index]
    }
}

// MARK: - メインアプリのContentView

struct ContentView: View {
    let todaysQuote = QuoteStore.todaysQuote()
    @State private var allQuotes = QuoteStore.quotes

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                // 今日の名言（ハイライト）
                VStack(spacing: 16) {
                    Text("今日の名言")
                        .font(.caption)
                        .foregroundStyle(.secondary)

                    Text("「\(todaysQuote.text)」")
                        .font(.title2)
                        .bold()
                        .multilineTextAlignment(.center)

                    Text("— \(todaysQuote.author)")
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                }
                .padding(24)
                .frame(maxWidth: .infinity)
                .background(
                    RoundedRectangle(cornerRadius: 16)
                        .fill(.blue.opacity(0.08))
                )
                .padding(.horizontal)

                // 全名言リスト
                List(allQuotes) { quote in
                    VStack(alignment: .leading, spacing: 4) {
                        Text(quote.text)
                            .font(.body)
                        Text("— \(quote.author)")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                    .padding(.vertical, 4)
                }
            }
            .navigationTitle("名言集")
        }
    }
}

#Preview {
    ContentView()
}


// ============================================
// ■ ウィジェット側のコード（自動生成された QuoteWidget.swift を "全置換"）
// ============================================
// ※ 下の /* ... */ を外し、自動生成ファイルの中身を全部消してから貼り付けます。
// ※ Quote と QuoteStore は手順3で QuoteStore.swift に移し、両ターゲットの
//    Target Membership に入れてあるので、ここでは再定義しません。
// ============================================

/*
import WidgetKit
import SwiftUI

// MARK: - タイムラインエントリ

struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

// MARK: - タイムラインプロバイダ

struct QuoteProvider: TimelineProvider {
    // プレースホルダー（読み込み中の仮表示）
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(id: 0, text: "読み込み中...", author: "")
        )
    }

    // スナップショット（ウィジェットギャラリーでのプレビュー）
    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote()
        )
        completion(entry)
    }

    // タイムライン（実際のウィジェット更新スケジュール）
    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
        let currentDate = Date()
        let quote = QuoteStore.todaysQuote()
        let entry = QuoteEntry(date: currentDate, quote: quote)

        // 次の日の0時にウィジェットを更新
        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
        )

        let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
        completion(timeline)
    }
}

// MARK: - ウィジェットのビュー

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }

    // 小サイズ
    var smallWidget: some View {
        VStack(spacing: 4) {
            Image(systemName: "quote.opening")
                .font(.caption)
                .foregroundStyle(.blue)

            Text(entry.quote.text)
                .font(.caption)
                .bold()
                .multilineTextAlignment(.center)
                .lineLimit(3)

            Text(entry.quote.author)
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
        .padding(12)
    }

    // 中サイズ
    var mediumWidget: some View {
        HStack(spacing: 16) {
            Image(systemName: "quote.opening")
                .font(.title)
                .foregroundStyle(.blue)

            VStack(alignment: .leading, spacing: 4) {
                Text("今日の名言")
                    .font(.caption2)
                    .foregroundStyle(.secondary)

                Text(entry.quote.text)
                    .font(.subheadline)
                    .bold()

                Text("— \(entry.quote.author)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }

            Spacer()
        }
        .padding()
    }
}

// MARK: - ウィジェット定義

@main
struct QuoteWidget: Widget {
    let kind: String = "QuoteWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: QuoteProvider()) { entry in
            QuoteWidgetEntryView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("今日の名言")
        .description("日替わりで名言を表示します")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}

// MARK: - プレビュー

#Preview(as: .systemMedium) {
    QuoteWidget()
} timeline: {
    QuoteEntry(date: .now, quote: QuoteStore.todaysQuote())
}
*/// ここに模範コード全体を貼る
```

**このアプリは何をするものか：**
このアプリは毎日1つの名言を表示するアプリである。
メインアプリでは今日の名言を大きく表示し、下には登録されているすべての名言を一覧で確認できる。また、ホーム画面に追加したウィジェットでも今日の名言を確認でき、日付が変わると自動で次の名言に更新される。
## コードの詳細解説

### TimelineProviderの仕組み

```swift
struct QuoteProvider: TimelineProvider {

    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(id: 0, text: "読み込み中...", author: "")
        )
    }

    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        completion(
            QuoteEntry(
                date: Date(),
                quote: QuoteStore.todaysQuote()
            )
        )
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {

        let currentDate = Date()
        let entry = QuoteEntry(
            date: currentDate,
            quote: QuoteStore.todaysQuote()
        )

        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
        )

        let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
        completion(timeline)
    }
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
TimelineProviderはウィジェットに表示する内容と、いつ更新するかを決めるクラスである。
placeholder()は読み込み中の表示を返し、getSnapshot()はウィジェット一覧に表示する内容を返す。getTimeline()では今日の名言を取得し、翌日の0時に更新するようタイムラインを作成している。
**なぜこう書くのか：**
ウィジェットは常に動作しているわけではないため、自分で更新のタイミングを指定する必要がある。今回は毎日名言が変わればよいので、翌日の0時に更新する設定にしている。
**もしこう書かなかったら：**
getTimeline()を書かなかった場合、ウィジェットは更新されず、同じ名言が表示され続ける。また、更新時刻を設定しなければ毎日の自動更新は行われない。
---

### TimelineEntryとウィジェットビュー

```swift
struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry

    var body: some View {
        Text(entry.quote.text)
    }
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
QuoteEntryはウィジェットに表示するデータをまとめる構造体である。
QuoteWidgetEntryViewは、そのデータを受け取り画面へ表示している。
**なぜこう書くのか：**
表示する内容をQuoteEntryにまとめることで、日付や名言など必要な情報を一つのオブジェクトとして扱えるため、コードが分かりやすくなる。
**もしこう書かなかったら：**
表示するデータを管理できず、ウィジェットへ名言を渡すことができないため、正しく表示されなくなる。
---

### ウィジェットサイズごとのレイアウト

```swift
@Environment(\.widgetFamily) var family

var body: some View {

    switch family {

    case .systemSmall:
        smallWidget

    case .systemMedium:
        mediumWidget

    default:
        mediumWidget
    }
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
ウィジェットのサイズを取得し、小サイズと中サイズで異なるレイアウトを表示している。
**なぜこう書くのか：**
ウィジェットのサイズによって表示できる情報量が異なるため、それぞれに最適なデザインを用意している。
**もしこう書かなかったら：**
すべて同じレイアウトになり、小さいサイズでは文字が切れたり、大きいサイズでは余白が多くなったりして見づらくなる。
---

### メインアプリとの連携

```swift
struct QuoteStore {

    static let quotes: [Quote] = [
        ...
    ]

    static func todaysQuote() -> Quote {

        let day = Calendar.current.ordinality(
            of: .day,
            in: .year,
            for: Date()
        ) ?? 0

        return quotes[day % quotes.count]
    }
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
名言データを一つのファイルで管理し、メインアプリとウィジェットの両方から同じデータを利用している。
**なぜこう書くのか：**
データを共通化することで、同じ内容を二重に書く必要がなくなり、修正も一か所だけで済む。
**もしこう書かなかったら：**
アプリとウィジェットで別々に名言を管理することになり、内容が一致しなくなる可能性がある。
---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API
| 項目                             | 説明                 | 使用例                                                  |
| ------------------------------ | ------------------ | ---------------------------------------------------- |
| `TimelineProvider`             | ウィジェットの更新タイミングを決める | `struct QuoteProvider: TimelineProvider`             |
| `TimelineEntry`                | ウィジェットへ渡すデータ       | `struct QuoteEntry: TimelineEntry`                   |
| `Timeline`                     | 更新スケジュールを作る        | `Timeline(entries:policy:)`                          |
| `Widget`                       | ウィジェット本体を定義する      | `struct QuoteWidget: Widget`                         |
| `@Environment(\.widgetFamily)` | ウィジェットサイズを取得する     | `switch family`                                      |
| `StaticConfiguration`          | 固定データのウィジェットを作る    | `StaticConfiguration(kind:provider:)`                |
| `Calendar`                     | 日付や曜日を取得する         | `Calendar.current.ordinality(...)`                   |
| `containerBackground`          | ウィジェット背景を設定する      | `.containerBackground(.fill.tertiary, for: .widget)` |



## 自分の実験メモ

**実験1：**
実験1
やったこと： supportedFamiliesから.systemMediumを削除した。
結果： 中サイズのウィジェットを追加できなくなった。
わかったこと： supportedFamiliesに書いたサイズだけが利用できる。

**実験2：**
やったこと： lineLimit(3)を削除した。
結果： 長い名言では文字が途中で見切れた。
わかったこと： 小さいウィジェットでは表示行数を制限することが重要である。

## AIに聞いて特に理解が深まった質問 TOP3
1. 質問
TimelineProviderは何のために使うのか。
得られた理解：
ウィジェットは通常のアプリのように常時動作しないため、表示内容と更新タイミングを管理する役割があることが分かった。
2. 質問
QuoteStoreを別ファイルにする理由は何か。
得られた理解：
メインアプリとウィジェットの両方で同じデータを利用するためであり、コードの重複を防げることが分かった。
3. 質問
WidgetFamilyとは何か。
得られた理解：
ウィジェットのサイズを取得し、それぞれに適したレイアウトへ切り替えるための仕組みであることが分かった。
## この章のまとめ
今回の学習では、WidgetKitを使ってホーム画面に表示できるウィジェットの作成方法を学んだ。TimelineProviderで更新スケジュールを管理し、TimelineEntryで表示するデータを渡す仕組みを理解できた。また、WidgetFamilyを利用することでサイズごとにレイアウトを切り替えられることや、QuoteStoreを共通化してメインアプリとウィジェットで同じデータを共有する方法も学んだ。これらの仕組みを利用することで、毎日自動更新される便利なウィジェットを作成できるようになった。
