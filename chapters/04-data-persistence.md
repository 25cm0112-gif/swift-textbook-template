# 第4章：データの永続化

> 執筆者：呉迎豪
> 最終更新：2026-5-27

## この章で学ぶこと
この章では、SwiftUIにおけるデータの永続化について学ぶ。具体的には、SwiftDataを利用したメモアプリを題材として、@Modelを使ったデータモデルの定義方法、modelContextによるデータの追加・削除・更新、@Queryを用いたデータの取得と画面への自動反映を実装する。また、@AppStorageを利用してユーザー設定を端末内に保存する方法についても学び、アプリを終了してもデータや設定を保持できる仕組みを理解する。
## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
import SwiftUI
import SwiftData

// MARK: - SwiftDataモデル

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}

// MARK: - アプリのエントリポイント
// ※ @main のあるAppファイルに以下を記述してください：
//
// @main
// struct MemoApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: Memo.self)
//     }
// }

// MARK: - メインビュー

struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
    @State private var isShowingAddSheet = false
    @State private var isShowingSettings = false

    var displayedMemos: [Memo] {
        if sortByFavorite {
            return memos.sorted { $0.isFavorite && !$1.isFavorite }
        }
        return memos
    }

    var body: some View {
        NavigationStack {
            Group {
                if memos.isEmpty {
                    ContentUnavailableView(
                        "メモがありません",
                        systemImage: "note.text",
                        description: Text("右上の＋ボタンからメモを追加してください")
                    )
                } else {
                    List {
                        ForEach(displayedMemos) { memo in
                            NavigationLink(destination: MemoEditView(memo: memo)) {
                                MemoRow(memo: memo)
                            }
                        }
                        .onDelete(perform: deleteMemos)
                    }
                }
            }
            .navigationTitle(userName.isEmpty ? "メモ帳" : "\(userName)のメモ帳")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button {
                        isShowingSettings = true
                    } label: {
                        Image(systemName: "gear")
                    }
                }
                ToolbarItem(placement: .topBarTrailing) {
                    Button {
                        isShowingAddSheet = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $isShowingAddSheet) {
                MemoAddView()
            }
            .sheet(isPresented: $isShowingSettings) {
                SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)
            }
        }
    }

    func deleteMemos(at offsets: IndexSet) {
        for index in offsets {
            let memo = displayedMemos[index]
            modelContext.delete(memo)
        }
    }
}

// MARK: - メモの行

struct MemoRow: View {
    let memo: Memo

    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(memo.title)
                    .font(.headline)

                Text(memo.content)
                    .font(.caption)
                    .foregroundStyle(.secondary)
                    .lineLimit(2)

                Text(memo.createdAt, style: .date)
                    .font(.caption2)
                    .foregroundStyle(.tertiary)
            }

            Spacer()

            if memo.isFavorite {
                Image(systemName: "star.fill")
                    .foregroundStyle(.yellow)
            }
        }
        .padding(.vertical, 2)
    }
}

// MARK: - メモ追加画面

struct MemoAddView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    @State private var title = ""
    @State private var content = ""

    var body: some View {
        NavigationStack {
            Form {
                Section("タイトル") {
                    TextField("メモのタイトル", text: $title)
                }
                Section("内容") {
                    TextEditor(text: $content)
                        .frame(minHeight: 200)
                }
            }
            .navigationTitle("新しいメモ")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        let memo = Memo(title: title, content: content)
                        modelContext.insert(memo)
                        dismiss()
                    }
                    .disabled(title.isEmpty)
                }
            }
        }
    }
}

// MARK: - メモ編集画面

struct MemoEditView: View {
    @Bindable var memo: Memo

    var body: some View {
        Form {
            Section("タイトル") {
                TextField("タイトル", text: $memo.title)
            }
            Section("内容") {
                TextEditor(text: $memo.content)
                    .frame(minHeight: 200)
            }
            Section {
                Toggle("お気に入り", isOn: $memo.isFavorite)
            }
        }
        .navigationTitle("メモを編集")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - 設定画面（AppStorageの活用）

struct SettingsView: View {
    @Binding var userName: String
    @Binding var sortByFavorite: Bool
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form {
                Section("ユーザー設定") {
                    TextField("あなたの名前", text: $userName)
                }
                Section("表示設定") {
                    Toggle("お気に入りを上に表示", isOn: $sortByFavorite)
                }
                Section {
                    Text("設定はアプリを閉じても保存されます")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .navigationTitle("設定")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("完了") { dismiss() }
                }
            }
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: Memo.self, inMemory: true)
}// ここに模範コード全体を貼る
```


**このアプリは何をするものか：
このアプリは、SwiftUIとSwiftDataを利用して作成したメモ帳アプリである。ユーザーはメモの追加・編集・削除を行うことができ、作成したメモは端末内に保存されるため、アプリを終了してもデータが消えない。また、お気に入り機能を使って重要なメモを管理したり、設定画面でユーザー名や表示順を保存したりできる。
さらに、@Queryによって保存されたメモ一覧が自動的に画面へ反映されるため、データ更新時に画面を手動で更新する必要がない。

## コードの詳細解説

### SwiftDataモデル（@Model）

```swift
@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
Memoというデータモデルを定義している。
タイトル、内容、作成日時、お気に入り状態を1つのメモとして管理している。
@Modelを付けることで、このクラスをSwiftDataで保存可能なデータとして扱えるようにしている。
**なぜこう書くのか：**
SwiftDataでは、永続化したいクラスに@Modelを付ける必要がある。
これによってSwiftDataが自動的にデータベース管理を行い、保存や取得を簡単にできる。

**もしこう書かなかったら：**
@Modelを付けなかった場合、SwiftDataがこのクラスを保存対象として認識できない。
その結果、@Queryで取得できなくなり、データ保存も行えなくなる。
---

### データの追加・削除（modelContext）

```swift
@Environment(\.modelContext) private var modelContextlet memo = Memo(title: title, content: content)
modelContext.insert(memo)
modelContext.delete(memo)// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
modelContextを利用して、SwiftDataに対してデータの追加や削除を行っている。
insert()で新しいメモを保存し、delete()で既存のメモを削除している。
**なぜこう書くのか：**
SwiftDataではmodelContextがデータベース操作を管理しているため、このオブジェクトを通してデータを操作する必要がある。
@Environmentを利用することで、現在の画面から簡単にSwiftDataへアクセスできる。
**もしこう書かなかったら：**
modelContextを使用しない場合、メモを追加しても保存されず、アプリを閉じると消えてしまう。
削除処理も行えなくなる。
---

### @Queryによるデータ取得

```swift
@Query(sort: \Memo.createdAt, order: .reverse)
private var memos: [Memo]// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
SwiftDataに保存されているMemoデータを取得している。
また、createdAtを基準に新しい順で並び替えている。
**なぜこう書くのか：**
@Queryを使うことで、データの変更を自動で監視し、画面へリアルタイムに反映できる。
そのため、メモ追加後に手動でリロードを書く必要がない。
**もしこう書かなかったら：**
@Queryを使わない場合、データ取得処理を自分で実装する必要がある。
また、データ更新時に画面が自動更新されなくなる。
---

### @AppStorageによる設定保存

```swift
@AppStorage("sortByFavorite")
private var sortByFavorite: Bool = false

@AppStorage("userName")
private var userName: String = ""// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
ユーザー設定を端末内へ保存している。
このコードでは、

お気に入り順表示
ユーザー名

を保存している。
**なぜこう書くのか：**
@AppStorageを使うと、簡単に設定情報を保存できる。
アプリを終了しても値が保持されるため、次回起動時にも同じ設定を利用できる。
**もしこう書かなかったら：**
普通の@Stateだけを使うと、アプリ終了時に設定が消えてしまう。
そのため、毎回ユーザー名を入力し直す必要がある。
---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目                       | 説明                        | 使用例                                    |
| ------------------------ | ------------------------- | -------------------------------------- |
| `@Model`                 | SwiftDataでデータを永続化するためのマクロ | `@Model class Memo {}`                 |
| `@Query`                 | SwiftDataからデータを取得する       | `@Query var memos: [Memo]`             |
| `modelContext`           | SwiftDataへの追加・削除を管理する     | `modelContext.insert(memo)`            |
| `@AppStorage`            | 設定情報を保存する                 | `@AppStorage("userName") var userName` |
| `NavigationStack`        | 画面遷移を管理する                 | `NavigationStack {}`                   |
| `NavigationLink`         | 別画面へ移動する                  | `NavigationLink(destination: ...)`     |
| `@Environment`           | 環境値を取得する                  | `@Environment(\.dismiss)`              |
| `@Bindable`              | SwiftDataモデルを編集可能にする      | `@Bindable var memo: Memo`             |
| `ContentUnavailableView` | データが空の時の表示を作る             | `ContentUnavailableView(...)`          |


## 自分の実験メモ
やったこと：
@AppStorageを@Stateへ変更してアプリを実行した。
結果：
アプリを閉じて再起動すると、入力したユーザー名や設定が消えた。
わかったこと：
@Stateは一時的な状態管理用であり、アプリ終了後には保存されないことが分かった。
設定を端末に保存したい場合は@AppStorageを使う必要がある。

**実験2：**
やったこと：
@Queryを削除してメモ一覧表示を試した。
結果：
保存したメモが画面に表示されなくなった。
わかったこと：
@QueryはSwiftDataからデータを取得し、画面へ自動反映する役割を持っていることが分かった。
また、データ変更時の自動更新にも必要だと理解できた。

## AIに聞いて特に理解が深まった質問 TOP3

1. 質問：
@Modelを付けると何が変わるのか？

得られた理解：
SwiftDataがクラスをデータベース用モデルとして認識し、自動で保存・取得できるようになることを理解した。

2. **質問：**
   なぜ@Queryを使うと画面が自動更新されるのか？
   **得られた理解：**
@QueryはSwiftDataの変更を監視しており、データ更新時にSwiftUIへ再描画を通知していることが分かった。
3. **質問：**@AppStorageと@Stateの違いは何か？
   **得られた理解：**
@Stateは画面内だけの一時データ、@AppStorageはアプリ終了後も保存される永続データという違いを理解した。
## この章のまとめ
この章では、SwiftUIで作成したアプリに対して、SwiftDataとAppStorageを利用したデータ保存機能を実装する方法を学んだ。
特に、@Modelによるデータモデル定義、modelContextを使った追加・削除処理、@Queryによるデータ取得と自動更新、そして@AppStorageによる設定保存の仕組みを理解できた。また、SwiftUIでは「状態管理」と「データ永続化」を組み合わせることで、実際のアプリらしい動作を簡単に実現できることを学んだ。
今後アプリ開発を行う際は、どのデータを一時的に扱い、どのデータを永続化するべきかを意識して設計したい。
（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
