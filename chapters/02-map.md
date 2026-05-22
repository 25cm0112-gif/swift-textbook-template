# 第2章：地図アプリの基本

> 執筆者：呉迎豪
> 最終更新：2026-5-20

## この章で学ぶこと
この章では、MapKit と CoreLocation を使って、現在地を取得しながら地図を表示する方法を学ぶ。さらに、周辺施設を検索して地図上にマーカーを表示し、カテゴリごとに検索を切り替える方法について理解する。

## 模範コードの全体像


```swift
import SwiftUI
import MapKit

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    let manager = CLLocationManager()
    var userLocation: CLLocationCoordinate2D?
    var authorizationStatus: CLAuthorizationStatus = .notDetermined

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
    }

    func requestPermission() {
        manager.requestWhenInUseAuthorization()
    }

    func startUpdating() {
        manager.startUpdatingLocation()
    }

    func stopUpdating() {
        manager.stopUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        userLocation = locations.last?.coordinate
    }

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        authorizationStatus = manager.authorizationStatus

        switch authorizationStatus {
        case .authorizedWhenInUse, .authorizedAlways:
            startUpdating()
        default:
            break
        }
    }
}

struct ContentView: View {
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var searchResults: [MKMapItem] = []
    @State private var selectedCategory: String = "コンビニ"

    let searchCategories = ["コンビニ", "カフェ", "レストラン", "駅"]

    var body: some View {
        ZStack(alignment: .top) {
            Map(position: $cameraPosition) {

                UserAnnotation()

                ForEach(searchResults, id: \.self) { item in
                    if let name = item.name {
                        Marker(name, coordinate: item.placemark.coordinate)
                    }
                }
            }

            VStack {
                ScrollView(.horizontal) {
                    HStack {
                        ForEach(searchCategories, id: \.self) { category in
                            Button(category) {
                                selectedCategory = category
                                Task {
                                    await searchNearby(query: category)
                                }
                            }
                        }
                    }
                }
                Spacer()
            }
        }
        .onAppear {
            locationManager.requestPermission()
        }
    }

    func searchNearby(query: String) async {
        guard let userLocation = locationManager.userLocation else { return }

        let request = MKLocalSearch.Request()
        request.naturalLanguageQuery = query

        request.region = MKCoordinateRegion(
            center: userLocation,
            span: MKCoordinateSpan(latitudeDelta: 0.02, longitudeDelta: 0.02)
        )

        do {
            let response = try await MKLocalSearch(request: request).start()
            searchResults = response.mapItems
        } catch {
            print(error.localizedDescription)
        }
    }
}// ここに模範コード全体を貼る
```

**このアプリは何をするものか：**

このアプリは、現在地を取得して、その周辺にあるコンビニやカフェなどの施設を検索する地図アプリである。カテゴリボタンを押すと、その種類の施設を検索し、検索結果を地図上にマーカーとして表示する。
## コードの詳細解説

### データモデル（ランドマーク構造体）

```swift
@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
位置情報の取得や、位置情報の許可状態を管理しているクラスである。

**なぜこう書くのか：**
CLLocationManagerDelegate を使うことで、位置情報が更新されたときに自動的に通知を受け取れる。また、@Observable を使うことで、位置情報の変化を SwiftUI に反映できる。

**もしこう書かなかったら：**
位置情報が変化しても画面が更新されず、現在地が取得できない。
---

### 地図の表示とカメラ制御

```swift
Map(position: $cameraPosition) {
    UserAnnotation()
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
MapKit を使って地図を表示している。また、UserAnnotation() によって現在地を地図上に表示している。
**なぜこう書くのか：**
cameraPosition を使うことで、地図の中心位置やズームを自由に制御できる。
**もしこう書かなかったら：**
地図は表示されても、現在地に移動できなかったり、地図の表示範囲を変更できなくなる。
---

### マーカーの表示

```swift
ForEach(searchResults, id: \.self) { item in
    if let name = item.name {
        Marker(name, coordinate: item.placemark.coordinate)
    }
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
検索結果を1件ずつ取り出して、地図上にマーカーを表示している。
**なぜこう書くのか：**
ForEach を使うことで、検索結果の数だけ自動的にマーカーを生成できる。
**もしこう書かなかったら：**
検索結果が複数あっても、地図上に表示できなくなる。
---

### フィルター機能

```swift
ForEach(searchCategories, id: \.self) { category in
    Button(category) {
        selectedCategory = category
        Task {
            await searchNearby(query: category)
        }
    }
}// 該当部分のコードを抜粋して貼る
```
何をしているか：

カテゴリボタンを押したときに、そのカテゴリ名で周辺検索を行っている。

なぜこう書くのか：

ForEach を使うことで、カテゴリ数が増えても簡単にボタンを生成できる。

もしこう書かなかったら：

カテゴリごとに個別のボタンを書く必要があり、コード量が増える。



## 新しく学んだSwiftの文法・API

| 項目                  | 説明              | 使用例                                    |
| ------------------- | --------------- | -------------------------------------- |
| `Map`               | SwiftUIで地図を表示する | `Map(position: $cameraPosition)`       |
| `Marker`            | 地図上にマーカーを表示する   | `Marker(name, coordinate: coordinate)` |
| `UserAnnotation`    | 現在地を表示する        | `UserAnnotation()`                     |
| `MKLocalSearch`     | 周辺施設を検索するAPI    | `MKLocalSearch(request: request)`      |
| `@Observable`       | 状態変化を監視する       | `@Observable class LocationManager`    |
| `Task`              | 非同期処理を実行する      | `Task { await searchNearby() }`        |
| `CLLocationManager` | 位置情報を取得する       | `CLLocationManager()`                  |


## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
やったこと：
検索カテゴリに「病院」を追加した。
結果：
周辺の病院が検索され、地図上に表示された。
わかったこと：
searchCategories に文字列を追加するだけで検索対象を増やせる

**実験2：**
やったこと：
latitudeDelta を 0.1 に変更した。
結果：
地図の表示範囲が広くなった。
わかったこと：
MKCoordinateSpan によってズーム倍率を調整できる。
## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   なぜ Task を使う必要があるのか？
   **得られた理解：**
searchNearby() は async 関数なので、非同期処理として実行する必要があるため。
3. **質問：**@Observable は何のために使うのか？
   **得られた理解：**データが変更されたとき、自動的に SwiftUI の画面を更新するため。

4. **質問：**MKLocalSearch はどうやって周辺施設を探しているのか？
   **得られた理解：**Apple Maps のデータベースを利用して、指定した地域とキーワードで検索している。

## この章のまとめ
この章では、MapKit と CoreLocation を使って、現在地の取得、地図表示、周辺施設検索を行う方法を学んだ。特に、Map、Marker、MKLocalSearch を組み合わせることで、実際の地図アプリに近い機能を実装できることが理解できた。また、async/await を使った非同期処理や @Observable による状態管理も重要なポイントだった。
（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
