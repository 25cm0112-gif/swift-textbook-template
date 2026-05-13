# 第2章：地図アプリの基本

> 執筆者：呉迎豪
> 最終更新：2025-5-13

## この章で学ぶこと
この章では、SwiftUI と MapKit を使って地図アプリの基本的な実装方法を学ぶ。ランドマークの表示やカテゴリによる絞り込み、選択した地点の詳細表示などを通して、地図UIの構築方法を理解する。また、状態管理やView分割など、SwiftUIらしい設計についても扱う。
## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
import SwiftUI
import MapKit
struct Landmark: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    static func == (lhs: Landmark, rhs: Landmark) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        var iconName: String {
            switch self {
            case .temple: return "building.columns"
            case .tower: return "antenna.radiowaves.left.and.right"
            case .park: return "leaf"
            }
        }

        var color: Color {
            switch self {
            case .temple: return .red
            case .tower: return .blue
            case .park: return .green
            }
        }
    }
}

// MARK: - サンプルデータ

extension Landmark {
    static let sampleData: [Landmark] = [
        Landmark(
            name: "浅草寺",
            description: "東京都内最古の寺院。雷門が有名。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7148, longitude: 139.7967),
            category: .temple
        ),
        Landmark(
            name: "東京タワー",
            description: "1958年に完成した高さ333mの電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6586, longitude: 139.7454),
            category: .tower
        ),
        Landmark(
            name: "東京スカイツリー",
            description: "高さ634mの世界一高い自立式電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7101, longitude: 139.8107),
            category: .tower
        ),
        Landmark(
            name: "明治神宮",
            description: "明治天皇と昭憲皇太后を祀る神社。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6764, longitude: 139.6993),
            category: .temple
        ),
        Landmark(
            name: "上野恩賜公園",
            description: "美術館や動物園がある広大な公園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7146, longitude: 139.7732),
            category: .park
        ),
        Landmark(
            name: "新宿御苑",
            description: "都心にある広さ58.3ヘクタールの庭園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6852, longitude: 139.7100),
            category: .park
        ),
    ]
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var cameraPosition: MapCameraPosition = .region(
        MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
            span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
        )
    )
    @State private var selectedLandmark: Landmark?
    @State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

    var filteredLandmarks: [Landmark] {
        Landmark.sampleData.filter { selectedCategories.contains($0.category) }
    }

    var body: some View {
        ZStack(alignment: .bottom) {
            // 地図
            Map(position: $cameraPosition, selection: $selectedLandmark) {
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                    .tag(landmark)
                }
            }
            .mapStyle(.standard(elevation: .realistic))

            // カテゴリフィルター
            VStack(spacing: 8) {
                if let landmark = selectedLandmark {
                    LandmarkCard(landmark: landmark)
                        .transition(.move(edge: .bottom))
                }

                CategoryFilter(selectedCategories: $selectedCategories)
            }
            .padding()
        }
        .onMapCameraChange { context in
            // 地図の操作に応じた処理を追加できる
        }
    }
}

// MARK: - カテゴリフィルター

struct CategoryFilter: View {
    @Binding var selectedCategories: Set<Landmark.Category>

    var body: some View {
        HStack(spacing: 8) {
            ForEach(Landmark.Category.allCases, id: \.self) { category in
                Button {
                    if selectedCategories.contains(category) {
                        selectedCategories.remove(category)
                    } else {
                        selectedCategories.insert(category)
                    }
                } label: {
                    HStack(spacing: 4) {
                        Image(systemName: category.iconName)
                        Text(category.rawValue)
                    }
                    .font(.caption)
                    .padding(.horizontal, 10)
                    .padding(.vertical, 6)
                    .background(
                        selectedCategories.contains(category)
                            ? category.color.opacity(0.2)
                            : Color.gray.opacity(0.1)
                    )
                    .foregroundStyle(
                        selectedCategories.contains(category)
                            ? category.color
                            : .gray
                    )
                    .clipShape(Capsule())
                }
            }
        }
        .padding(8)
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 16))
    }
}

// MARK: - ランドマーク詳細カード

struct LandmarkCard: View {
    let landmark: Landmark

    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            HStack {
                Image(systemName: landmark.category.iconName)
                    .foregroundStyle(landmark.category.color)
                Text(landmark.name)
                    .font(.headline)
                Spacer()
            }
            Text(landmark.description)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding()
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}

#Preview {
    ContentView()
}// ここに模範コード全体を貼る
```

**このアプリは何をするものか：**
このアプリは、東京周辺のランドマークを地図上にピンで表示し、カテゴリ（寺社・タワー・公園）ごとに表示／非表示を切り替えられる地図アプリです。
ピンをタップすると、その場所の名前と簡単な説明がカードとして画面下に表示され、どんな場所かすぐに確認できます。
また、地図を動かしながら興味のあるスポットを探したり、カテゴリフィルターで見たい種類の場所だけに絞り込むことができます。
## コードの詳細解説
### データモデル（ランドマーク構造体）

```swift
import SwiftUI
import MapKit

// MARK: - データモデル

struct Landmark: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        var iconName: String {
            switch self {
            case .temple: return "building.columns"
            case .tower: return "antenna.radiowaves.left.and.right"
            case .park: return "leaf"
            }
        }

        var color: Color {
            switch self {
            case .temple: return .red
            case .tower: return .blue
            case .park: return .green
            }
        }
    }
}
```

**何をしているか：**
このコードでは、地図アプリで扱う「ランドマーク（場所情報）」のデータ構造を定義している。各ランドマークは、名前・説明・座標・カテゴリを持ち、1つのまとまったデータとして管理できるようになっている。また、カテゴリは寺社・タワー・公園の3種類に分けられており、それぞれアイコンや色も一緒に定義することで、地図上の表示に使いやすい形にしている。
**なぜこう書くのか：**
ランドマークの情報（名前・場所・種類など）をひとつの構造体にまとめることで、データを整理しやすくし、アプリ全体で一貫して扱えるようにするためです。Identifiable に準拠させることで、SwiftUIのリスト表示や地図のマーカー表示で自動的にデータを識別でき、コードをシンプルに保てます。また、カテゴリごとにアイコンや色を持たせることで、表示処理を別で書かずに済み、見た目とデータをセットで管理できるようになります。

**もしこう書かなかったら：**
ランドマークの情報がバラバラに管理されてしまい、地図表示や一覧表示のたびに複数の変数を個別に扱う必要が出てきて、コードが複雑になる。また、SwiftUIの ForEach や Map のような仕組みでデータを扱う際に識別ができず、表示や更新がうまくいかない可能性がある。さらに、カテゴリごとのアイコンや色も別々に管理することになり、重複コードが増えて保守性が下がってしまう。

---

### 地図の表示とカメラ制御

```swift
@State private var cameraPosition: MapCameraPosition = .region(
    MKCoordinateRegion(
        center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
        span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
    )
)

@State private var selectedLandmark: Landmark?
@State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

var filteredLandmarks: [Landmark] {
    Landmark.sampleData.filter { selectedCategories.contains($0.category) }
}

var body: some View {
    ZStack(alignment: .bottom) {
        // 地図
        Map(position: $cameraPosition, selection: $selectedLandmark) {
            ForEach(filteredLandmarks) { landmark in
                Marker(
                    landmark.name,
                    systemImage: landmark.category.iconName,
                    coordinate: landmark.coordinate
                )
                .tint(landmark.category.color)
                .tag(landmark)
            }
        }
        .mapStyle(.standard(elevation: .realistic))
    }
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
このコードでは、Map を使って地図を表示し、ランドマークをマーカーとして地図上に配置している。また cameraPosition によって地図の中心位置やズーム範囲を管理し、ユーザーの操作や選択に応じて表示範囲を制御できるようにしている。さらに、選択されたカテゴリに応じて表示するランドマークを絞り込み、必要な情報だけを地図に表示している。
**なぜこう書くのか：**
cameraPosition で地図の状態を管理することで、SwiftUIの状態管理と連動した自然なカメラ操作が可能になるためである。また、filteredLandmarks を使ってデータを絞り込むことで、表示ロジックをシンプルに保ちつつ、ユーザーの操作に応じて動的に地図の内容を変更できるようにしている。
**もしこう書かなかったら：**
地図の表示範囲や中心位置を細かく制御できず、ユーザー操作に応じたズームや移動が難しくなる。また、カテゴリフィルターがない場合はすべてのランドマークが常に表示されてしまい、情報量が多すぎて見づらくなる。さらに、表示ロジックが分散するとコードが複雑になり、状態管理も難しくなる。
---

### マーカーの表示

```swift
Map(position: $cameraPosition, selection: $selectedLandmark) {
    ForEach(filteredLandmarks) { landmark in
        Marker(
            landmark.name,
            systemImage: landmark.category.iconName,
            coordinate: landmark.coordinate
        )
        .tint(landmark.category.color)
        .tag(landmark)
    }
}
.mapStyle(.standard(elevation: .realistic))
```

**何をしているか：**
このコードでは、ForEach を使ってランドマークの配列を順番に取り出し、それぞれを Marker として地図上に表示している。各マーカーには名前、アイコン、座標が設定され、さらにカテゴリに応じて色分けされることで視覚的に区別できるようになっている。
**なぜこう書くのか：**
データ配列をそのままUIに反映できる形にすることで、ランドマークの追加・削除・変更があっても自動的に表示が更新されるようにするためである。また、ForEach を使うことでコードの重複を避け、宣言的に「データからUIを作る」SwiftUIの考え方に沿った実装になる。
**もしこう書かなかったら：**
マーカーを1つずつ手動で書く必要があり、データが増えるほどコードが長くなって管理が難しくなる。また、データの変更をUIに反映する仕組みを別で作る必要が出てしまい、バグが起きやすくなったり保守性が大きく下がってしまう。
---

### フィルター機能

```swift
@State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

var filteredLandmarks: [Landmark] {
    Landmark.sampleData.filter { selectedCategories.contains($0.category) }
}

struct CategoryFilter: View {
    @Binding var selectedCategories: Set<Landmark.Category>

    var body: some View {
        HStack(spacing: 8) {
            ForEach(Landmark.Category.allCases, id: \.self) { category in
                Button {
                    if selectedCategories.contains(category) {
                        selectedCategories.remove(category)
                    } else {
                        selectedCategories.insert(category)
                    }
                } label: {
                    HStack(spacing: 4) {
                        Image(systemName: category.iconName)
                        Text(category.rawValue)
                    }
                }
            }
        }
    }
}
```

**何をしているか：**
このコードでは、選択されたカテゴリだけを表示するためのフィルター機能を実装している。selectedCategories でユーザーが選んだカテゴリを管理し、その条件に合うランドマークだけを filteredLandmarks で抽出して地図に表示している。また、ボタン操作によってカテゴリの追加・削除を切り替えられるUIも作っている。
**なぜこう書くのか：**
Set を使うことで複数選択の管理がシンプルになり、検索や追加・削除の処理も効率よく行えるためである。また、filteredLandmarks を計算プロパティにすることで、状態が変わるたびに自動で表示内容が更新され、UIとデータの整合性を保ちやすくしている。
**もしこう書かなかったら：**
カテゴリごとの表示制御を手動で行う必要があり、条件分岐が増えてコードが複雑になる。また、表示と状態管理が分離されず、どのデータが表示対象なのか分かりにくくなる。結果としてバグが発生しやすくなり、機能追加や修正も難しくなる。
---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Map` | SwiftUIで地図を表示するビューコンポーネント | `Map(position: .constant(.region(region)))` |
| 例：`Marker` | 地図上に位置をマーキングするコンポーネント | `Marker("名前", coordinate: coordinate)` |
| | | |
| | | |
| | | |
| 項目                       | 説明                        | 使用例                                                        |
| ------------------------ | ------------------------- | ---------------------------------------------------------- |
| `Map`                    | SwiftUIで地図を表示するビューコンポーネント | `Map(position: $cameraPosition)`                           |
| `Marker`                 | 地図上に位置情報を表示するマーカー         | `Marker(landmark.name, coordinate::landmark.coordinate)`   |
| `MapCameraPosition`      | 地図の中心位置やズーム範囲を管理する        | `@State private var cameraPosition: MapCameraPosition`     |
| `MKCoordinateRegion`     | 地図の表示範囲を設定する構造体           | `MKCoordinateRegion(center: ..., span: ...)`               |
| `CLLocationCoordinate2D` | 緯度・経度を表す座標データ             | `CLLocationCoordinate2D(latitude: 35.0, longitude: 139.0)` |
| `ForEach`                | 配列データを繰り返し表示する            | `ForEach(filteredLandmarks) { landmark in in ... }`        |
| `@State`                 | View内で状態を保持し、変更時にUIを更新する  | `@State private var selectedLandmark: Landmark?`           |
| `@Binding`               | 親Viewの状態を子Viewから共有・更新する   | `@Binding var selectedCategories`                          |
| `filter`                 | 条件に合うデータだけを抽出する           | `sampleData.filter { ... }`                                |
| `Set`                    | 重複しないデータ集合を管理する           | `Set(Landmark.Category.allCases)`                          |
| `.mapStyle()`            | 地図の表示スタイルを変更する            | `.mapStyle(.standard(elevation: .realistic))`              |
| `.tint()`                | マーカーやUI部品の色を設定する          | `.tint(landmark.category.color)`                           |

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
