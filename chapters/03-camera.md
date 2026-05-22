# 第3章：カメラの利用

> 執筆者：呉迎豪
> 最終更新：2026-5-22

## この章で学ぶこと
この章では、PhotosPicker を使って端末内の写真を選択し、CoreImage のフィルターを適用する方法を学ぶ。さらに、SwiftUI を使った画像表示や、加工した画像をフォトライブラリへ保存する方法について理解する。
## 模範コードの全体像



```swift
import SwiftUI
import PhotosUI
import Photos
import CoreImage
import CoreImage.CIFilterBuiltins

// MARK: - フィルター定義

enum PhotoFilter: String, CaseIterable, Identifiable {
    case original = "オリジナル"
    case sepia = "セピア"
    case mono = "モノクロ"
    case chrome = "クローム"
    case fade = "フェード"
    case bloom = "ブルーム"

    var id: String { rawValue }

    func apply(to inputImage: CIImage, context: CIContext) -> CIImage? {
        switch self {
        case .original:
            return inputImage
        case .sepia:
            let filter = CIFilter.sepiaTone()
            filter.inputImage = inputImage
            filter.intensity = 0.8
            return filter.outputImage
        case .mono:
            let filter = CIFilter.photoEffectMono()
            filter.inputImage = inputImage
            return filter.outputImage
        case .chrome:
            let filter = CIFilter.photoEffectChrome()
            filter.inputImage = inputImage
            return filter.outputImage
        case .fade:
            let filter = CIFilter.photoEffectFade()
            filter.inputImage = inputImage
            return filter.outputImage
        case .bloom:
            let filter = CIFilter.bloom()
            filter.inputImage = inputImage
            filter.radius = 10
            filter.intensity = 0.8
            return filter.outputImage
        }
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var originalUIImage: UIImage?
    @State private var displayImage: Image?
    @State private var currentFilter: PhotoFilter = .original
    @State private var isSaving = false
    @State private var showSaveAlert = false
    @State private var saveMessage = ""

    private let context = CIContext()

    var body: some View {
        NavigationStack {
            VStack(spacing: 16) {
                // 画像表示
                if let image = displayImage {
                    image
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .frame(maxHeight: 350)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                        .padding(.horizontal)
                } else {
                    placeholderView
                }

                // フィルター選択
                if originalUIImage != nil {
                    filterSelector
                }

                // ボタン群
                HStack(spacing: 16) {
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選ぶ", systemImage: "photo")
                    }
                    .buttonStyle(.bordered)

                    if displayImage != nil {
                        Button {
                            saveFilteredImage()
                        } label: {
                            Label("保存", systemImage: "square.and.arrow.down")
                        }
                        .buttonStyle(.borderedProminent)
                        .disabled(isSaving)
                    }
                }
                .padding()

                Spacer()
            }
            .navigationTitle("フォトフィルター")
            .onChange(of: selectedItem) { _, newItem in
                Task { await loadOriginalImage(from: newItem) }
            }
            .onChange(of: currentFilter) { _, _ in
                applyFilter()
            }
            .alert("保存結果", isPresented: $showSaveAlert) {
                Button("OK") {}
            } message: {
                Text(saveMessage)
            }
        }
    }

    // MARK: - プレースホルダー

    private var placeholderView: some View {
        RoundedRectangle(cornerRadius: 12)
            .fill(.gray.opacity(0.1))
            .frame(height: 300)
            .overlay {
                VStack(spacing: 8) {
                    Image(systemName: "camera.filters")
                        .font(.system(size: 48))
                        .foregroundStyle(.gray)
                    Text("写真を選んでフィルターを試そう")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .padding(.horizontal)
    }

    // MARK: - フィルター選択UI

    private var filterSelector: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack(spacing: 12) {
                ForEach(PhotoFilter.allCases) { filter in
                    VStack(spacing: 4) {
                        // フィルタープレビュー（サムネイル）
                        if let thumbnail = createThumbnail(filter: filter) {
                            Image(uiImage: thumbnail)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 60, height: 60)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                                .overlay(
                                    RoundedRectangle(cornerRadius: 8)
                                        .stroke(
                                            currentFilter == filter ? Color.blue : Color.clear,
                                            lineWidth: 3
                                        )
                                )
                        }

                        Text(filter.rawValue)
                            .font(.caption2)
                            .foregroundStyle(
                                currentFilter == filter ? .blue : .secondary
                            )
                    }
                    .onTapGesture {
                        currentFilter = filter
                    }
                }
            }
            .padding(.horizontal)
        }
    }

    // MARK: - 画像処理

    func loadOriginalImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                originalUIImage = uiImage
                currentFilter = .original
                displayImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像読み込みエラー: \(error)")
        }
    }

    func applyFilter() {
        guard let uiImage = originalUIImage,
              let ciImage = CIImage(image: uiImage) else { return }

        guard let outputImage = currentFilter.apply(to: ciImage, context: context) else { return }

        if let cgImage = context.createCGImage(outputImage, from: ciImage.extent) {
            // 元画像の scale と imageOrientation を引き継がないと
            // EXIF回転情報が落ちて画像が上下反転して表示される
            displayImage = Image(uiImage: UIImage(cgImage: cgImage,
                                                  scale: uiImage.scale,
                                                  orientation: uiImage.imageOrientation))
        }
    }

    func createThumbnail(filter: PhotoFilter) -> UIImage? {
        guard let uiImage = originalUIImage,
              let ciImage = CIImage(image: uiImage) else { return nil }

        guard let output = filter.apply(to: ciImage, context: context) else { return nil }

        if let cgImage = context.createCGImage(output, from: ciImage.extent) {
            return UIImage(cgImage: cgImage,
                           scale: uiImage.scale,
                           orientation: uiImage.imageOrientation)
        }
        return nil
    }

    func saveFilteredImage() {
        guard let uiImage = originalUIImage,
              let ciImage = CIImage(image: uiImage),
              let output = currentFilter.apply(to: ciImage, context: context),
              let cgImage = context.createCGImage(output, from: ciImage.extent) else { return }

        // PHAssetChangeRequest は imageOrientation を尊重して保存するため、
        // ここで orientation を渡しておかないと保存ファイルも上下反転する
        let finalImage = UIImage(cgImage: cgImage,
                                 scale: uiImage.scale,
                                 orientation: uiImage.imageOrientation)
        isSaving = true

        // PHPhotoLibrary を使うと、保存の成否をコールバックで受け取れる
        PHPhotoLibrary.shared().performChanges {
            PHAssetChangeRequest.creationRequestForAsset(from: finalImage)
        } completionHandler: { success, error in
            DispatchQueue.main.async {
                isSaving = false
                if success {
                    saveMessage = "写真を保存しました"
                } else {
                    saveMessage = "保存に失敗しました\n\(error?.localizedDescription ?? "原因不明のエラー")"
                }
                showSaveAlert = true
            }
        }
    }
}

#Preview {
    ContentView()
}// ここに模範コード全体を貼る
```

**このアプリは何をするものか：**
このアプリは、iPhone 内の写真を選択して、セピアやモノクロなどのフィルターを適用できる画像加工アプリである。加工後の画像はフォトライブラリへ保存することもできる。
## コードの詳細解説

### PhotosPickerによる写真選択

```swift
enum PhotoFilter: String, CaseIterable, Identifiable {
    case original = "オリジナル"
    case sepia = "セピア"
    case mono = "モノクロ"
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
使用するフィルターの種類を enum で定義している。

**なぜこう書くのか：**
enum を使うことで、使えるフィルターを安全に管理できる。また CaseIterable によって全フィルターを一覧表示できる。

**もしこう書かなかったら：**
文字列だけで管理すると、入力ミスや存在しないフィルター名を使ってしまう可能性がある。
---

### 画像の非同期読み込み

```swift
func apply(to inputImage: CIImage, context: CIContext) -> CIImage?// 該当部分のコードを抜粋して貼る
```
何をしているか：

選択されたフィルターを画像へ適用している。

なぜこう書くのか：

CoreImage の CIFilter を使うことで、高品質な画像加工を簡単に実現できる。

もしこう書かなかったら：

画像加工機能を実装できず、元画像しか表示できない。

### UIViewControllerRepresentableによるカメラ連携

```swift
PhotosPicker(selection: $selectedItem, matching: .images)// 該当部分のコードを抜粋して貼る
```

何をしているか：

フォトライブラリから画像を選択する画面を表示している。

なぜこう書くのか：

PhotosPicker は SwiftUI 向けに用意された写真選択コンポーネントであり、簡単に画像選択機能を実装できる。

もしこう書かなかったら：

ユーザーが写真を読み込めなくなる。
---

### Coordinatorパターン

```swift
if let image = displayImage {
    image
        .resizable()
        .aspectRatio(contentMode: .fit)
}// 該当部分のコードを抜粋して貼る
```
何をしているか：

選択した画像や加工後の画像を画面に表示している。

なぜこう書くのか：

.resizable() を使うことで画像サイズを変更でき、.aspectRatio(.fit) によって縦横比を保ったまま表示できる。

もしこう書かなかったら：

画像サイズが崩れたり、一部が切れてしまう可能性がある。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目                | 説明                | 使用例                          |
| ----------------- | ----------------- | ---------------------------- |
| `PhotosPicker`    | 写真選択UI            | `PhotosPicker(selection:)`   |
| `CIFilter`        | 画像フィルター           | `CIFilter.sepiaTone()`       |
| `CIImage`         | CoreImage用画像      | `CIImage(image: uiImage)`    |
| `CIContext`       | 画像レンダリング管理        | `CIContext()`                |
| `PHPhotoLibrary`  | 写真保存API           | `PHPhotoLibrary.shared()`    |
| `Task`            | 非同期処理             | `Task { await loadImage() }` |
| `CaseIterable`    | enum全件取得          | `PhotoFilter.allCases`       |
| `Image(uiImage:)` | UIImageをSwiftUI表示 | `Image(uiImage: uiImage)`    |


## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
やったこと：
sepia の intensity を 1.0 に変更した。
結果：
セピア色がより強くなった。
わかったこと：
フィルターのパラメータによって画像効果を細かく調整できる。
**実験2：**
やったこと：
blur フィルターを追加した。
let filter = CIFilter.gaussianBlur()
結果：
ぼかし効果を追加できた。
わかったこと：
CoreImage には多くの画像加工フィルターが用意されている。
## AIに聞いて特に理解が深まった質問 TOP3

1. 質問：
なぜ Task を使う必要があるのか？

得られた理解：
画像読み込みは時間がかかるため、非同期処理にして UI が止まらないようにする必要がある。

2. 質問：
CIImage と UIImage の違いは？

得られた理解：
UIImage は画面表示用、CIImage は画像加工用として使われる。

3. 質問：
なぜ imageOrientation を引き継ぐ必要があるのか？

得られた理解：
iPhone写真には回転情報があり、それを保持しないと画像の向きが崩れるため。

## この章のまとめ
この章では、PhotosPicker を使った写真選択、CoreImage を使った画像加工、そしてフォトライブラリへの保存方法について学んだ。特に、CIFilter を利用すると少ないコードで本格的な画像加工ができることが理解できた。また、Task を使った非同期処理や EXIF 回転情報の扱いなど、実際のアプリ開発で重要なポイントも学べた。
（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
