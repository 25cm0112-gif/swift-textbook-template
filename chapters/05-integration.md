# 第5章：機能統合の実践

> 執筆者：呉迎豪
> 最終更新：2026-6-12

## この章で学ぶこと
この章では、SwiftUI・SwiftData・MapKit・PhotosPickerを組み合わせて、写真と位置情報を保存・表示するアプリを作成した。単独の機能だけでなく、複数の機能を連携させて一つのアプリとして動作させる方法を学んだ。また、データの永続化や画面遷移についても理解を深めた。
（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

例：この章では、これまでに学んだカメラ・地図・データ保存の各機能を組み合わせて、「フォトマップ」アプリを実装する方法を学ぶ。具体的には撮影した写真をGPS位置情報と一緒に保存し、地図上に表示し、永続化したデータを検索・編集するアプリを題材にする。複数機能を統合するためのアーキテクチャ設計が重要になる。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
import SwiftUI
import SwiftData
import MapKit
import PhotosUI

// MARK: - データモデル

@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

// MARK: - アプリエントリポイント
// ※ App ファイルに以下を記述：
//
// @main
// struct PhotoMapApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: PhotoRecord.self)
//     }
// }

// MARK: - メインビュー（タブ構成）

struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}

// MARK: - マップタブ

struct MapTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var records: [PhotoRecord]
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var isShowingAddSheet = false
    @State private var selectedRecord: PhotoRecord?

    var body: some View {
        NavigationStack {
            ZStack(alignment: .bottomTrailing) {
                Map(position: $cameraPosition) {
                    UserAnnotation()

                    ForEach(records) { record in
                        Annotation(record.title, coordinate: record.coordinate) {
                            Button {
                                selectedRecord = record
                            } label: {
                                if let uiImage = record.uiImage {
                                    Image(uiImage: uiImage)
                                        .resizable()
                                        .aspectRatio(contentMode: .fill)
                                        .frame(width: 40, height: 40)
                                        .clipShape(Circle())
                                        .overlay(Circle().stroke(.white, lineWidth: 2))
                                        .shadow(radius: 2)
                                } else {
                                    Image(systemName: "photo.circle.fill")
                                        .font(.title)
                                        .foregroundStyle(.blue)
                                }
                            }
                        }
                    }
                }
                .mapControls {
                    MapUserLocationButton()
                }

                // 追加ボタン
                Button {
                    isShowingAddSheet = true
                } label: {
                    Image(systemName: "plus.circle.fill")
                        .font(.system(size: 56))
                        .foregroundStyle(.blue)
                        .background(Circle().fill(.white))
                        .shadow(radius: 4)
                }
                .padding(24)
            }
            .navigationTitle("フォトマップ")
            .sheet(isPresented: $isShowingAddSheet) {
                AddRecordView(locationManager: locationManager)
            }
            .sheet(item: $selectedRecord) { record in
                RecordDetailView(record: record)
            }
        }
    }
}

// MARK: - 一覧タブ

struct ListTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \PhotoRecord.createdAt, order: .reverse) private var records: [PhotoRecord]

    var body: some View {
        NavigationStack {
            List {
                ForEach(records) { record in
                    HStack(spacing: 12) {
                        if let uiImage = record.uiImage {
                            Image(uiImage: uiImage)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 50, height: 50)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                        }

                        VStack(alignment: .leading, spacing: 4) {
                            Text(record.title)
                                .font(.headline)
                            Text(record.createdAt, style: .date)
                                .font(.caption)
                                .foregroundStyle(.secondary)
                        }
                    }
                }
                .onDelete { offsets in
                    for index in offsets {
                        modelContext.delete(records[index])
                    }
                }
            }
            .navigationTitle("記録一覧")
        }
    }
}

// MARK: - 記録追加画面

struct AddRecordView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    let locationManager: LocationManager

    @State private var title = ""
    @State private var memo = ""
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImageData: Data?
    @State private var previewImage: Image?

    var body: some View {
        NavigationStack {
            Form {
                Section("写真") {
                    if let image = previewImage {
                        image
                            .resizable()
                            .aspectRatio(contentMode: .fit)
                            .frame(maxHeight: 200)
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                    }

                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選択", systemImage: "photo")
                    }
                }

                Section("情報") {
                    TextField("タイトル", text: $title)
                    TextField("メモ（任意）", text: $memo, axis: .vertical)
                        .lineLimit(3...6)
                }

                Section("位置情報") {
                    if let location = locationManager.currentLocation {
                        Text("緯度: \(location.latitude, specifier: "%.4f")")
                        Text("経度: \(location.longitude, specifier: "%.4f")")
                    } else {
                        Text("位置情報を取得中...")
                            .foregroundStyle(.secondary)
                    }
                }
            }
            .navigationTitle("新しい記録")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        saveRecord()
                    }
                    .disabled(title.isEmpty || locationManager.currentLocation == nil)
                }
            }
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    if let data = try? await newItem?.loadTransferable(type: Data.self) {
                        selectedImageData = data
                        if let uiImage = UIImage(data: data) {
                            previewImage = Image(uiImage: uiImage)
                        }
                    }
                }
            }
        }
    }

    func saveRecord() {
        guard let location = locationManager.currentLocation else { return }

        let record = PhotoRecord(
            title: title,
            memo: memo,
            latitude: location.latitude,
            longitude: location.longitude,
            imageData: selectedImageData
        )
        modelContext.insert(record)
        dismiss()
    }
}

// MARK: - 記録詳細画面

struct RecordDetailView: View {
    let record: PhotoRecord

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                if let uiImage = record.uiImage {
                    Image(uiImage: uiImage)
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                }

                VStack(alignment: .leading, spacing: 8) {
                    Text(record.title)
                        .font(.title2)
                        .bold()

                    if !record.memo.isEmpty {
                        Text(record.memo)
                            .foregroundStyle(.secondary)
                    }

                    Text(record.createdAt, style: .date)
                        .font(.caption)
                        .foregroundStyle(.tertiary)
                }
                .frame(maxWidth: .infinity, alignment: .leading)

                // ミニマップ
                Map {
                    Marker(record.title, coordinate: record.coordinate)
                }
                .frame(height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 12))
            }
            .padding()
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: PhotoRecord.self, inMemory: true)
}// ここに模範コード全体を貼る
```

**このアプリは何をするものか：**
このアプリは、撮影または選択した写真を現在地の位置情報と一緒に保存できるフォトマップアプリである。
保存した記録は
地図上にマーカーとして表示される
一覧画面で確認できる
詳細画面で写真・メモ・位置情報を確認できる
さらに、保存したデータはSwiftDataによってアプリを終了しても保持される。
（アプリの動作を自分の言葉で説明する。スクリーンショットを貼ってもよい。）

## コードの詳細解説

### データモデルの設計

```swift
@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
写真のタイトル・メモ・位置情報・画像データ・作成日時を一つのデータとして管理している。
（この部分が果たしている役割を説明する）

**なぜこう書くのか：**
@Modelを付けることでSwiftDataがこのクラスをデータベースとして扱えるようになる。
画像はUIImageではなくData型で保存することで、SwiftDataへ保存できるようになっている。
（別の書き方ではなく、この書き方が選ばれている理由を説明する）

**もしこう書かなかったら：**
@Modelを付けなければSwiftDataへ保存できず、アプリを終了すると記録がすべて消えてしまう。
（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）

---

### タブ構成の設計

```swift
TabView {
    MapTab()
        .tabItem {
            Label("マップ", systemImage: "map")
        }

    ListTab()
        .tabItem {
            Label("一覧", systemImage: "list.bullet")
        }
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
画面を「マップ」と「一覧」の2つに分け、タブで切り替えられるようにしている。
**なぜこう書くのか：**
一つの画面にすべての機能を入れると見づらくなるため、用途ごとに画面を分けることで操作しやすくなる。
**もしこう書かなかったら：**
地図と一覧が同じ画面に表示されるため画面が複雑になり、使いにくいアプリになってしまう。
---

### カメラと位置情報の連携

```swift
guard let location = locationManager.currentLocation else {
    return
}

let record = PhotoRecord(
    title: title,
    memo: memo,
    latitude: location.latitude,
    longitude: location.longitude,
    imageData: selectedImageData
)// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
現在取得しているGPS情報を写真と一緒に保存している。
**なぜこう書くのか：**
写真だけでは撮影場所が分からないため、位置情報も保存することで地図上に表示できるようになる。
**もしこう書かなかったら：**
位置情報が保存されないので、マップ上にマーカーを表示できなくなる。
---

### SwiftDataでの画像保存

```swift
modelContext.insert(record)// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
作成したPhotoRecordをSwiftDataへ保存している。
**なぜこう書くのか：**
SwiftDataを利用することでデータベースへの保存を簡単に行うことができ、アプリを終了してもデータを保持できる。
**もしこう書かなかったら：**
データはメモリ上だけに存在するため、アプリを閉じると記録が消えてしまう。
---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目                      | 説明                    | 使用例                               |
| ----------------------- | --------------------- | --------------------------------- |
| `@Model`                | SwiftDataのデータモデルを作成する | `@Model class PhotoRecord`        |
| `@Query`                | SwiftDataからデータを取得する   | `@Query var records`              |
| `TabView`               | タブ画面を作成する             | `TabView { ... }`                 |
| `Map`                   | 地図を表示する               | `Map { ... }`                     |
| `Annotation`            | 地図上へマーカーを表示する         | `Annotation(...)`                 |
| `PhotosPicker`          | 写真ライブラリから画像を選択する      | `PhotosPicker(...)`               |
| `LocationManager`       | 現在地を取得する              | `manager.startUpdatingLocation()` |
| `modelContext.insert()` | データを保存する              | `modelContext.insert(record)`     |
| `sheet()`               | モーダル画面を表示する           | `.sheet(...)`                     |
| `NavigationStack`       | 画面遷移を管理する             | `NavigationStack { ... }`         |


## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：タイトルを入力しない状態で保存ボタンを押した。
- 結果：保存ボタンが押せなかった。
- わかったこと：.disabled(title.isEmpty)によって入力チェックが行われていることが分かった。

**実験2：**
- やったこと：位置情報の取得を許可しない状態でアプリを起動した。
- 結果：「位置情報を取得中...」と表示され、保存できなかった。
- わかったこと：GPS情報が取得できないと写真を保存できない仕様になっていることが分かった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**なぜ画像をUIImageではなくData型で保存するのか？
   **得られた理解：**SwiftDataはUIImageを直接保存できないため、Data型へ変換して保存し、表示するときにUIImageへ戻していることを理解した。

2. **質問：**@Modelとは何か？
   **得られた理解：**SwiftDataから保存済みデータを自動取得し、データが追加・削除されると画面も自動更新されることを理解した。

3. **質問：**
   **得られた理解：**

## この章のまとめ
この章では、SwiftUIだけではなく、SwiftData・MapKit・PhotosPicker・位置情報APIを組み合わせて一つのアプリを作成した。特に、データモデルを設計し、写真・位置情報・メモをまとめて保存する仕組みや、保存したデータを地図や一覧へ自動表示する方法を学んだ。今後アプリを開発する際にも、複数の機能を連携させる設計方法を活用できると感じた。
（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
