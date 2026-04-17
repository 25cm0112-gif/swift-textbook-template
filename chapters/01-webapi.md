# 第1章：WebAPIの基本

> 執筆者：呉迎豪
> 最終更新：2027-4-17

## この章で学ぶこと
この章では、SwiftUIを使った音楽検索アプリの基本的な作り方を学ぶ。
iTunes APIを利用してデータを取得し、非同期通信（async/await）で画面に表示する方法を扱う。
また、検索機能・ローディング表示・リスト表示などの基本的なUI構成についても理解する。

## 模範コードの全体像
import SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var songs: [Song] = []
    @State private var searchText: String = ""
    @State private var isLoading: Bool = false

    var body: some View {
        NavigationStack {
            VStack {
                // 検索バー
                HStack {
                    TextField("アーティスト名を入力", text: $searchText)
                        .textFieldStyle(.roundedBorder)

                    Button("検索") {
                        Task {
                            await searchMusic()
                        }
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(searchText.isEmpty)
                }
                .padding(.horizontal)

                // 検索結果リスト
                if isLoading {
                    ProgressView("検索中...")
                        .padding()
                    Spacer()
                } else if songs.isEmpty {
                    ContentUnavailableView(
                        "曲を検索してみよう",
                        systemImage: "music.note",
                        description: Text("アーティスト名を入力して検索ボタンを押してください")
                    )
                } else {
                    List(songs) { song in
                        SongRow(song: song)
                    }
                }
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - API通信

    func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.3)
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)

                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }
        }
        .padding(.vertical, 4)
    }
}

#Preview {
    ContentView()
}
（教員から配布された模範コードをここに貼り付ける）

```swift
// ここに模範コード全体を貼る
```

**このアプリは何をするものか：**
このアプリは、アーティスト名を入力して検索すると、そのアーティストに関連する楽曲を一覧で表示するアプリである。
検索結果はiTunesのAPIから取得され、曲名・アーティスト名・ジャケット画像がリスト形式で表示される。
また、通信中はローディング表示が出るため、検索の進行状況も分かるようになっている。
## コードの詳細解説
### データモデル（Codable構造体）
役割
APIから返ってくる「全体の箱」
results の中に曲データの配列が入っている

```swift
struct SearchResponse: Codable {
    let results: [Song]
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
（この部分が果たしている役割を説明する）

**なぜこう書くのか：**
（別の書き方ではなく、この書き方が選ばれている理由を説明する）
① JSON構造と一致させるため
→ decodeを自動で成功させるため
② データの役割を分けるため
→ 「箱」と「中身」を分離
③ SwiftUIで扱いやすくするため
→ ListでSongだけ使える
**もしこう書かなかったら：**
（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）
let response = try JSONDecoder().decode([Song].self, from: data)
デコードエラーになる可能性が高い
APIは { results: [...] } という形なのに、配列だと誤解するため
songs が空になる
画面に何も表示されない
---

### API通信の処理

```swift
guard let encodedText = searchText.addingPercentEncoding(
    withAllowedCharacters: .urlQueryAllowed
) else { return }
// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
この関数は、ユーザーが入力したアーティスト名を使ってiTunes APIにリクエストを送り、検索結果を取得して画面に表示する処理です。
**なぜこう書くのか：**
① asyncを使う理由
ネット通信は時間がかかるため、画面を止めないように非同期で実行する必要があるからです。
② URLエンコードする理由
日本語（例：米津玄師など）はそのままだとURLとして使えないため、正しい形式に変換する必要があります。
③ guard letを使う理由
不正なURLや変換失敗が起きた場合に、処理を安全に止めるためです。
④ isLoadingを使う理由
通信中であることをUIに伝え、「検索中…」などの表示を出すためです。
⑤ JSONDecoderを使う理由
APIから返ってくるJSONデータをSwiftで扱える型（Song）に変換するためです。
**もしこう書かなかったら：**

---

### ビューの構成

```swift
if isLoading
else if songs.isEmpty
else// 該当部分のコードを抜粋して貼る
```

**何をしているか：**

このビューは、音楽検索アプリの画面全体を構成しており、以下の3つの役割を持っています。
検索バー（入力＋ボタン）
検索状態の表示（読み込み・空・結果）
検索結果のリスト表示
つまり、ユーザーの操作と表示をまとめて管理するメイン画面です。

**なぜこう書くのか：**

① NavigationStackを使う理由
画面のタイトル表示や、将来的な画面遷移（詳細画面など）に対応するためです。
② VStackを使う理由
検索バーと結果表示を「縦に並べる」ための基本レイアウトだからです。
③ 検索バーをHStackにする理由
TextField（入力欄）
Button（検索ボタン）
を横並びにして、自然な検索UIにするためです。
④ 状態によって表示を分ける理由
if isLoading
else if songs.isEmpty
else
ユーザーに「今何が起きているか」を明確に伝えるためです。
通信中 → ProgressView
結果なし → 案内画面
結果あり → リスト表示
⑤ Listを使う理由
大量の曲データを効率よくスクロール表示するためです。SwiftUIが自動で最適化してくれます。
⑥ SongRowに分ける理由
UIを分割することで：
コードが読みやすくなる
再利用しやすくなる
役割が明確になる

**もしこう書かなかったら：**

① NavigationStackがない場合
タイトルが表示されない
画面遷移ができない構造になる
② VStack / HStackを使わない場合
UIが崩れる
要素が意図しない配置になる
③ 状態分岐がない場合
ローディング中でもリストが空で表示される
何も起きていないように見える
UXが悪くなる
④ Listを使わない場合
手動でスクロール処理が必要になる
大量データでパフォーマンスが落ちる
⑤ SongRowに分けない場合
ContentViewが長くなりすぎる
保守性が下がる
バグ修正が難しくなる
---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Codable` | JSONデータとSwiftの構造体を相互変換するプロトコル | `struct Song: Codable { ... }` |
| 例：`async/await` | 非同期処理を同期的に書ける構文 | `let data = try await URLSession.shared.data(from: url)` |
| | | |
| | | |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：
- 結果：
- わかったこと：

**実験2：**
- やったこと：
- 結果：
- わかったこと：

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   **得られた理解：**

2. **質問：**
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
